---
title:  "[토이 프로젝트] Lock으로 인한 응답 지연 이슈 : 로직 개선"

categories:
  - Database
tags:
  - Database

toc: true
toc_sticky: true

date: 2023-10-25
---

# 들어가며

저번 [포스팅](https://cire0304.github.io/database/title-%ED%86%A0%EC%9D%B4-%ED%94%84%EB%A1%9C%EC%A0%9D%ED%8A%B8-%EC%9D%91%EB%8B%B5-%EC%A7%80%EC%97%B0-%EC%9D%B4%EC%8A%88-%EC%9D%B8%EB%8D%B1%EC%8A%A4/)에서는 클라이언트의 응답이 늦어지는 것을 `connection-pool-size` 조정 및 인덱스를 활용하여, 서버의 퍼포먼스를 향상시키는 것을 다루었습니다.

하지만 DB connetion이 많아져 `Lock`이 걸린 레코드에 접근하는 트랜잭션려는 트랜잭션이 많아져, 클라이언트에 대한 응답 시간이 늘어나는 추가적인 이슈가 발생했습니다.

![](../../image/Toy-Project/[토이%20프로젝트]%20풀스캔%20이슈%20:%20인덱스/락%20쿼리%20성능.png)

본 포스팅에서는 제가 어떻게 위와 같이 `Lock`으로 인한 성능 저하를 최소화하는 방법에 대해서 써보려 합니다.

# 현재의 코드

## 테이블

현재 프로젝트의 데이터베이스 테이블 구성은 아래와 같습니다.
- 쿠폰 : 쿠폰에 대한 정보를 담고있는 테이블
- 사용자 쿠폰 : 사용자가 발급받은 쿠폰에 대한 정보를 담고 있는 테이블

![](../../image/Toy-Project/[토이%20프로젝트]%20Lock으로%20인한%20응답%20지연%20이슈%20:%20로직%20개선/../[토이%20프로젝트]%20Lock으로%20인한%20응답%20지연%20이슈%20:%20로직%20개선/Lock으로%20인한%20응답%20지연%20현상%20-%201.png)

위의 그림에서 사용자 쿠폰은 쿠폰에 대한 PK를 FK로 가지고 있는 상황입니다.


## 사용자 쿠폰 발급 로직

현재 사용자 쿠폰 발급관련 코드는 아래와 같습니다. (예시를 위해 코드를 간략화 하였습니다.)

```java
============ 엔티티 ============

@Entity
@Table(name = "user_coupon")
public class UserCoupon extends BaseEntity {

    @Id
    private Long userCouponId;

    @ManyToOne
    @JoinColumn(name = "coupon_id")
    private Coupon coupon;

    ... 코드 생략
}

===

@Entity
@Table(name = "coupon")
public class Coupon extends BaseEntity {

    @Id
    private Long couponId;

    @Column(name = "left_quantity")
    private Long leftQuantity;

    public void decrease() {
      if (leftQuantity <= 0) throw new IllegalStateException("쿠폰의 재고가 없습니다.");
      leftQuantity--;
    }

    ... 코드 생략
}

============ 서비스 로직 ============

@Service
@RequiredArgsConstructor
public class IssueUserCouponService {

  private final CouponRepository couponRepository;
  private final UserCouponRepository userCouponRepository;

  @Transactional
  private UserCoupon issueUserCoupon(IssueUserCouponCommand command) {
    LocalDateTime currentTime = LocalDateTime.now();

    // 쿠폰 과발급 방지를 위해, 비관적 락을 이용한 조회
    Coupon coupon = couponRepository.findByIdWithPessimisticLock(command.couponId());
      
    // 쿠폰 재고 감소
    coupon.decrease(currentTime);

    // 사용자 쿠폰 발급 및 아이디 반환
    UserCoupon issuedUserCoupon = new UserCoupon(command.userId(), coupon, currentTime);
    return userCouponRepository.save(issuedUserCoupon).getUserCouponId();
  }
}
```

사용자 쿠폰을 발급하기 위해서는 쿠폰에 대한 정보가 필요하기 때문에 위의 서비스 로직에서 쿠폰을 조회하였습니다.  
또한 아래의 그림과 같이 쿠폰이 과발급 되는 것을 막기위해 `비관적 락`을 사용하여 조회 후, 쿠폰의 재고를 감소하였습니다.

![](../../image/Spring/2023-10-1-title:%20동시성%20이슈/1.png)

# 문제 상황

그렇다면 위의 서비스 로직에서 문제가 되는 점을 무엇일까요?  
그건 바로 너무 넓은 트랜잭션 범위에서 `Lock`을 가진 쿠폰이 사용되고 있다는 점입니다.

![](../../image/Toy-Project/[토이%20프로젝트]%20Lock으로%20인한%20응답%20지연%20이슈%20:%20로직%20개선/../[토이%20프로젝트]%20Lock으로%20인한%20응답%20지연%20이슈%20:%20로직%20개선/Lock으로%20인한%20응답%20지연%20현상%20-%202.png)

위의 그림에서 볼 수 있듯이 동시성을 제어하기 위해 `비관적 락`을 통해 쿠폰을 조회하게 된다면, 트랜잭션이 끝날 때까지 다른 트랜잭션은 쿠폰을 조회할 수 없게됩니다.  
즉, 클라이언트 요청 하나가 끝날때까지 다른 클라이언트의 요청을 수행할 수 없습니다.

# Lock을 가진 트랜잭션의 범위를 줄이는 방법

동시성을 제어하기 위해 `Lock`을 사용했습니다. 하지만 `Lock`을 사용함으로써, 요청에 대한 처리 속도가 감소했습니다.  
`Lock`으로 인해 성능이 낮아졌다면, 딱 필요한 만큼만 `Lock`을 사용하는 것은 어떨까요?

방법은 아래의 코드와 같습니다.

```java

@Service
@RequiredArgsConstructor
public class IssueUserCouponFacade {

    private final CouponQuantityService couponQuantityService;
    private final IssueUserCouponService issueUserCouponService;


    // @Transactoinal 어노테이션 주석 처리
    public Long issue(IssueUserCouponCommand command) {

        // 쿠폰 재고 감소
        couponQuantityService.decrease(command);

        // 사용자 쿠폰 발급 및 사용자 쿠폰 아이디 반환
        return issueUserCouponService.issue(command);
    }
}

===

@Service
@RequiredArgsConstructor
public class IssueUserCouponService {

    private final CouponRepository couponRepository;
    private final UserCouponRepository userCouponRepository;

    public Long issue(IssueUserCouponCommand command) {
      
      // ================= 중요 ===================
      // Lock을 사용하지 않은 쿠폰 조회 
      Coupon coupon = couponRepository.findById(command.couponId())
             .orElseThrow(NotFoundCouponException::new);
      // =========================================

      UserCoupon issuedUserCoupon = new UserCoupon(command.userId(), coupon);
      return userCouponRepository.save(issuedUserCoupon).getUserCouponId();
    }
}

===

@RequiredArgsConstructor
public class CouponQuantityService {

    private final CouponRepository couponRepository;

    public void decrease(IssueUserCouponCommand command) {

      // ================= 매우 중요 ===================
      // 쿼리 결과에 영향을 받은 레코드 수 반환
      // 네이티브 쿼리 사용
      int result = couponRepository.decrease(command.couponId());
      // ==============================================

      // 결과에 영향을 받은 레코드가 1이 아니면 예외 발생
      if (result != 1) throw new IllegalStateException("쿠폰의 재고가 없습니다.");
    }
}

===

public interface CouponRepository extends JpaRepository<Coupon, Long> {

    // 남은 재고가 1 이상인 쿠폰일 경우, 쿠폰 재고 감소
    @Transactional
    @Modifying(clearAutomatically = true)
    @Query(value =  "update coupon " +
                    "set left_quantity = left_quantity - 1 " +
                    "where coupon_id = :couponId and left_quantity > 0 "
                    , nativeQuery = true)
    int decrease(Long couponId);
}
```

위의 코드를 요약하면 아래의 그림과 같습니다.

![](../../image/Toy-Project/[토이%20프로젝트]%20Lock으로%20인한%20응답%20지연%20이슈%20:%20로직%20개선/../[토이%20프로젝트]%20Lock으로%20인한%20응답%20지연%20이슈%20:%20로직%20개선/Lock으로%20인한%20응답%20지연%20현상%20-%203.png)

쿠폰의 재고 감소를 위해 `Lock`을 사용한 쿠폰 조회을 사용하지 않았습니다.  
그 대신, 남은 재고가 1 이상인 쿠폰이면 업데이트하는 네이티브 쿼리를 사용하여 쿠폰 재고 감소 로직을 수행하였습니다.  
그리고 네이티브 쿼리 결과에 따라 쿠폰 발급 성공 여부를 판단합니다. (위 코드의 네이티브 쿼리 반환값은 쿼리에 의해 영향을 받은 레코드 수입니다.)


# 로직 개선 후 결과

위의 코드 처럼 로직을 개선할 경우, `Lock`으로 인한 성능 저하를 최소화 하였고, 성능(TPS)을 약 36% 향상 시킬 수 있었습니다.

![](../../image/Toy-Project/[토이%20프로젝트]%20Lock으로%20인한%20응답%20지연%20이슈%20:%20로직%20개선/../[토이%20프로젝트]%20Lock으로%20인한%20응답%20지연%20이슈%20:%20로직%20개선/Lock으로%20인한%20응답%20지연%20현상%20-%204.png)


# 정리

`Lock`은 동시성을 제어할 수 있도록 도와주는 훌륭한 기술이지만, 성능 저하 이슈가 존재합니다.  
따라서 `Lock`을 사용해야 한다면 최소한으로 사용하는 것이 좋을 것 같습니다.

# 추가 이슈

쿠폰 재고 감소 트랜잭션과 사용자 쿠폰 발급 트랜잭션을 별도의 트랜잭션에서 실행하도록 하여 성능을 높였습니다.

하지만, 아래의 이슈가 존재합니다.

- 쿠폰 재고 감소 트랜잭션은 성공
- 하지만 사용자 쿠폰 트랜잭션이 실패할 경우 어떻게 복구할 것인지?

다음 포스팅은 위의 이슈를 해결하는 글을 작성하려 합니다.  
끝까지 봐주셔서 감사합니다!
