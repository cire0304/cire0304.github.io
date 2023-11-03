---
title:  "[토이 프로젝트] Lock으로 인한 응답 지연 이슈 : Lock 경합 최소화"

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

본 포스팅에서는 위와 같이 `Lock 경합`으로 인한 성능 저하를 최소화하는 방법에 대해서 써보려 합니다.

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

    @Column(name = "coupon_id")
    private Long couponId;

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

    // 쿠폰 과발급 방지를 위해, 비관적 락을 이용한 조회
    Coupon coupon = couponRepository.findByIdWithPessimisticLock(command.couponId());
      
    // 쿠폰 재고 감소
    coupon.decrease();

    // 사용자 쿠폰 발급 및 아이디 반환
    UserCoupon issuedUserCoupon = new UserCoupon(command.userId(), coupon.getCouponId);
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



## 방법 1 : 외래키 제약 사용 X

사용자 테이블은 쿠폰에 대한 외래키를 가지고 있었습니다. 외래키는 두 개의 테이블을 연결해주는 연결 다리 역할을 하고, 새롭게 추가되는 행에서 외래키에 해당하는 값이 참조하는 테이블에 존재하는지를 체크합니다. 그리고 이 부분에서 성능의 병목점이 발생합니다.

"외래키는 부모테이블이나 자식 테이블에 데이터가 있는지 체크하는 작업이 필요하므로 잠금이 여러 테이블로 전파되고, 그로인해 데드락이 발생할 수 있다. 그래서 실무에서는 잘 사용하지 않는다."
-Real MySQL 3장-
{: .notice--info}

사용자 쿠폰발급을 위해 쓰기 쿼리가 수행될 때, 쿠폰 테이블의 해당 행을 `공유 락`을 사용하여 데이터 정합성을 확인합니다.  
`Lock`을 사용하기 때문에, `비관적 락`을 사용하는 쿠폰 발급 로직에서 `Lock 경합`이 발생하여 성능 이슈가 발생한 것입니다.

## 방법 2 : Lock 사용 범위 최소화

동시성을 제어하기 위해 `Lock`을 사용했습니다. 하지만 `Lock`을 사용함으로써, 요청에 대한 처리 속도가 감소했습니다.  
`Lock`으로 인해 성능이 낮아졌다면, 딱 필요한 만큼만 `Lock`을 사용하는 것은 어떨까요?

Lock 사용 범위 최소화하는 방법은 아래의 코드와 같습니다.

```java

@Service
@RequiredArgsConstructor
public class IssueUserCouponFacade {

    private final CouponQuantityService couponQuantityService;
    private final IssueUserCouponService issueUserCouponService;


    @Transactoinal
    public Long issue(IssueUserCouponCommand command) {

      // 사용자 쿠폰 발급
      UserCoupon issuedCoupon = new UserCoupon(command.userId(), command.couponId());
      Long issuedUserCouponId = userCouponRepository.save(issuedCoupon)
                                                    .getUserCouponId();

      // 쿠폰 과발급 방지를 위해, 비관적 락을 이용한 조회
      Coupon coupon = couponRepository.findByIdWithPessimisticLock(command.couponId());

      // 쿠폰 재고 감소
      coupon.decrease();

      // 사용자 쿠폰 아이디 반환
      return issuedUserCouponId
    }
}
```

위의 코드는 기존의 코드와 무엇이 달라졌을까요?  
요약하면 아래의 그림과 같습니다.

![](../../image/Toy-Project/[토이%20프로젝트]%20Lock으로%20인한%20응답%20지연%20이슈%20:%20로직%20개선/../[토이%20프로젝트]%20Lock으로%20인한%20응답%20지연%20이슈%20:%20로직%20개선/Lock으로%20인한%20응답%20지연%20현상%20-%203.png)

위의 그림처럼 사용자 쿠폰 발급 로직과 쿠폰 재고 감소 로직의 순서를 바꿈으로써, 트랜잭션이 `Lock`을 점유하고 있는 시간을 최소화하여, `Lock 경합`을 최소화할 수 있습니다.

# 결과

테이블에서 외래키를 제거하고, 위의 코드 처럼 로직을 개선하여 `Lock`으로 인한 성능 저하를 최소화 하였고, 성능(TPS)을 약 22% 향상 시킬 수 있었습니다.

![](../../image/Toy-Project/[토이%20프로젝트]%20Lock으로%20인한%20응답%20지연%20이슈%20:%20로직%20개선/../[토이%20프로젝트]%20Lock으로%20인한%20응답%20지연%20이슈%20:%20로직%20개선/Lock으로%20인한%20응답%20지연%20현상%20-%204.png)


# 마무리

`Lock`은 동시성을 제어할 수 있도록 도와주는 훌륭한 기술이지만, 성능 저하 이슈가 존재합니다.  
따라서 `Lock`을 사용해야 한다면 최소한으로 잘 사용하는 것이 좋을 것 같습니다.

끝까지 봐주셔서 감사합니다!
