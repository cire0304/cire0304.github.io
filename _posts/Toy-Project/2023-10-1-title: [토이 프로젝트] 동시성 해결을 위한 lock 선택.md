---
title:  "[토이 프로젝트] 동시성 해결을 위한 lock 선택"

categories:
  - Spring
tags:
  - [Spring, Java]

toc: true
toc_sticky: true

date: 2023-10-01
---

## 들어가며

지난 [포스팅]()에서는 `@Transactional`을 사용할 때 발생할 수 있는 데드락에 대해서 다루었습니다.  
이번 포스팅에서는 `동시성 이슈`을 해결하기 위한 다양한 방법에 대해서 알아보려 합니다.

동시성을 제어할 수 있는 방법은 다음과 같습니다.

- `synchronized` 키워드를 이용한 동시성 제어
- `Pessimistic Lock`을 이용한 동시성 제어
- `Optimistic Lock`을 이용한 동시성 제어
- `Named Lock`을 이용한 동시성 제어

각 방법의 특징을 살펴보고, `쿠폰 발급 기능`을 구현할 때 어떤 방법이 적합한지 알아보겠습니다.

## 엔티티 코드

```java
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@Entity
@Table(name = "coupon")
public class Coupon {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long couponId;

    @Column(name = "quantity")
    private Long quantity;

    public void decrease() {
        if (quantity <= 0) throw new IllegalStateException("쿠폰의 재고가 없습니다.");
        quantity--;
    }

}
```

## synchronized을 이용한 동시성 제어

동시성 문제는 **두 개 이상의 스레드가 동일한 자원에 동시에 접근**할 때 발생합니다.  
따라서 데이터에 접근할 때 **한번에 하나의 스레드(트랜잭션)이 접근**할 수 있도록 `synchronized` 키워드를 통해 동시성 문제를 해결할 수 있습니다.

```java
@RequiredArgsConstructor
@Service
public class IssueCouponService {

    private final CouponRepository couponRepository;
    private final UserCouponRepository userCouponRepository;

    public synchronized void issue(Long couponId) {
        // 등록된 쿠폰 조회
        Coupon coupon = couponRepository.findById(couponId)
                .orElseThrow(RuntimeException::new);

        coupon.decrease(); // 쿠폰 재고 감소
        couponRepository.saveAndFlush(coupon); // DB 반영

        UserCoupon issuedCoupon = new UserCoupon(coupon); // 사용자 쿠폰 발급
        userCouponRepository.saveAndFlush(issuedCoupon); // DB 반영
    }

}
```

> **실제 쿼리**  

```text
// 등록된 쿠폰 조회
Id Command    Argument
129 Query	set session transaction read only
129 Query	SET autocommit=0
129 Query	select c1_0.coupon_id,c1_0.quantity from coupon c1_0 where c1_0.coupon_id=1
129 Query	commit
129 Query	SET autocommit=1

// 쿠폰 재고 감소 및 DB 반영
129 Query	set session transaction read write
129 Query	SET autocommit=0
129 Query	select c1_0.coupon_id,c1_0.quantity from coupon c1_0 where c1_0.coupon_id=1
129 Query	update coupon set quantity=1 where coupon_id=1
129 Query	commit
129 Query	SET autocommit=1

// 사용자 쿠폰 발급 및 DB 반영
129 Query	SET autocommit=0
129 Query	insert into user_coupon (coupon_id) values (1)
129 Query	commit
129 Query	SET autocommit=1
```

하지만 위의 코드는 다수의 서버(프로세스)에서 쿠폰 발급 서비스를 제공할 시, **동시성을 제어하지 못합니다.**
  - `synchronized`의 적용 범위는 하나의 프로세스로 한정되기 때문입니다.

![](../../image/Spring/2023-10-1-title:%20동시성%20이슈/6.png)

## `Pessimistic Lock`을 이용한 동시성 제어

`Pessimistic Lock`을 사용한 동시성 제어는 테이블의 row에 접근시, `Lock`을 걸고 다른 `Lock`이 걸려 있지 않을 경우에만 수정을 가능하게 할 수 있습니다.

```java
@RequiredArgsConstructor
@Service
public class OptimisticLockIssueCouponService {

    private final CouponRepository couponRepository;
    private final UserCouponRepository userCouponRepository;

    @Transactional
    public void issue(Long couponId) {
        Coupon coupon = couponRepository.findByIdWithOptimisticLock(couponId); // 비관적 락을 이용한 조회

        coupon.decrease();
        UserCoupon issuedCoupon = new UserCoupon(coupon);
        userCouponRepository.save(issuedCoupon);
    }

}

===

public interface CouponRepository extends JpaRepository<Coupon, Long> {

    @Lock(value = LockModeType.PESSIMISTIC_WRITE)
    @Query("select c from Coupon c where c.couponId=:couponId")
    Coupon findByIdWithPessimisticLock(Long couponId);

}
```

> **실제 쿼리**  

```text
Id Command    Argument
209 Query	SET autocommit=0
210 Query	SET autocommit=0

// select 구문에 for update 을 추가하여 coupon 테이블의 해당 레코드에 Exclusive lock
210 Query	select c1_0.coupon_id,c1_0.quantity from coupon c1_0 where c1_0.coupon_id=1 for update // for update 추가
209 Query	select c1_0.coupon_id,c1_0.quantity from coupon c1_0 where c1_0.coupon_id=1 for update // for update 추가

// 첫 번째 쿠폰 발급 요청
209 Query	insert into user_coupon (coupon_id) values (1)
209 Query	update coupon set quantity=1 where coupon_id=1
209 Query	commit
209 Query	SET autocommit=1

// 두 번째 쿠폰 발급 요청
210 Query	insert into user_coupon (coupon_id) values (1)
210 Query	update coupon set quantity=0 where coupon_id=1
210 Query	commit
210 Query	SET autocommit=1
```

## `Optimistic Lock`을 이용한 동시성 제어  

`Optimistic Lock`은 락을 사용하지 않고, 수정할 때 내가 먼저 이 값을 수정했다고 명시하여 다른 사람이 동일한 조건으로 값을 수정할 수 없게 하는 것입니다.
JPA의 `@Version`을 이용해서 쉽게 구현할 수 있습니다.

```java
@RequiredArgsConstructor
@Service
public class OptimisticLockIssueCouponService {

    private final CouponRepository couponRepository;
    private final UserCouponRepository userCouponRepository;

    @Transactional
    public void issue(Long couponId) {
        Coupon coupon = couponRepository.findByIdWithOptimisticLock(couponId); // 낙관적 락을 이용한 조회

        coupon.decrease();
        UserCoupon issuedCoupon = new UserCoupon(coupon);
        userCouponRepository.save(issuedCoupon);
    }

}

===

public interface CouponRepository extends JpaRepository<Coupon, Long> {

    @Lock(value = LockModeType.OPTIMISTIC)
    @Query("select c from Coupon c where c.couponId=:couponId")
    Coupon findByIdWithOptimisticLock(Long couponId);
    
}

===

@NoArgsConstructor(access = AccessLevel.PROTECTED)
@Entity
@Table(name = "coupon")
public class Coupon {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long couponId;

    @Column(name = "quantity")
    private Long quantity;

    @Version
    private Long version; // 낙관적 락을 위한 버전 추가

    public void decrease() {
        if (quantity <= 0) throw new IllegalStateException("쿠폰의 재고가 없습니다.");
        quantity--;
    }

}
```

`낙관적 락`을 이용한 동시성 제어는, 실패 케이스 핸들링을 개발자가 직접 해주어야 하는 번거러움이 존재합니다.
따라서, 쿠폰 발급에 실패한 케이스일 때, 재시도를 할 수 있도록 아래와 같이 코드를 짤 수 있습니다.

```java
@RequiredArgsConstructor
@Service
public class OptimisticLockCouponFacade {

    private final OptimisticLockIssueCouponService optimisticLockIssueCouponService;

    public void issue(Long couponId) throws InterruptedException {
      // 쿠폰 발급 실패시, 재시도 로직
      while (true) {
          try {
              optimisticLockIssueCouponService.issue(couponId);

              break;
          }  catch (Exception e) {
              Thread.sleep(10);
          }
      }
    }
}
```

> **실제 쿼리**  


```text
Id Command    Argument
250 Query	SET autocommit=0
249 Query	SET autocommit=0

// 쿠폰 발급 동시 요청
249 Query	select c1_0.coupon_id,c1_0.quantity,c1_0.version from coupon c1_0 where c1_0.coupon_id=1
250 Query	select c1_0.coupon_id,c1_0.quantity,c1_0.version from coupon c1_0 where c1_0.coupon_id=1
250 Query	insert into user_coupon (coupon_id) values (1)
249 Query	insert into user_coupon (coupon_id) values (1)
250 Query	update coupon set quantity=1,version=2 where coupon_id=1 and version=1 // coupon version 1 => 2 업데이트
249 Query	update coupon set quantity=1,version=2 where coupon_id=1 and version=1 
250 Query	select version as version_ from coupon where coupon_id=1
250 Query	commit
250 Query	SET autocommit=1

// 다른 트랜잭션에서 이미 coupon version을 업데이트 했기 때문에, 쿠폰 발급 실패
249 Query	rollback
249 Query	SET autocommit=1

// 쿠폰 발급 재시도
249 Query	SET autocommit=0
249 Query	select c1_0.coupon_id,c1_0.quantity,c1_0.version from coupon c1_0 where c1_0.coupon_id=1
249 Query	insert into user_coupon (coupon_id) values (1)
249 Query	update coupon set quantity=0,version=3 where coupon_id=1 and version=2 coupon version 2 => 3 업데이트
249 Query	select version as version_ from coupon where coupon_id=1
249 Query	commit
249 Query	SET autocommit=1
```

위의 쿼리에서 볼 수 있다시피, version에 의해서 쿠폰 발급에 실패하면, 재시도를 하는 것을 확인할 수 있습니다.

## `Named Lock`을 이용한 동시성 제어

`Named Lock`은 테이블이나 레코드, 데이터베이스 객체가 아닌 사용자가 지정한 문자열에 대해 락을 획득하고 반납하는 잠금으로, 한 세션이 `Lock`을 획득한다면, 다른 세션은 해당 세션이 `Lock`을 해제한 이후 획득할 수 있습니다.

`Lock`을 획득하고 반납할 수 있도록 `Lock` 레포지토리를 만들도록 하겠습니다.

```java
// 편의성을 위해 JPA의 Nativive Query 기능을 활용하여 구현하였습니다.
// 또한, 예제를 간단히 하기 위해 Coupon 엔티티를 사용하였습니다.
public interface LockRepository extends JpaRepository<Coupon, Long> {

    @Query(value = "select get_lock(:key, 3000)", nativeQuery = true)
    void getLock(String key);

    @Query(value = "select release_lock(:key)", nativeQuery = true)
    void releaseLock(String key);

}
```

아래와 같이 락을 획득하고 반납할 수 있도록 코드를 구현했습니다.

```java
@RequiredArgsConstructor
@Service
public class NamedLockCouponFacade {

    private final LockRepository lockRepository;

    private final IssueCouponService issueCouponService;

    @Transactional
    public void issue(Long couponId) {
        try {
            lockRepository.getLock(couponId.toString());
            issueCouponService.issue(couponId);
        }finally {
            lockRepository.releaseLock(couponId.toString());
        }
    }

}

===

@RequiredArgsConstructor
@Service
public class IssueCouponService {

    private final CouponRepository couponRepository;
    private final UserCouponRepository userCouponRepository;

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void issue(Long couponId) {
        Coupon coupon = couponRepository.findById(couponId)
                .orElseThrow(RuntimeException::new);

        coupon.decrease(); // 쿠폰 재고 감소
        UserCoupon issuedCoupon = new UserCoupon(coupon);
        userCouponRepository.save(issuedCoupon); // 사용자 쿠폰 발급
    }

}
```

> **실제 쿼리**  


```text
290 Query	SET autocommit=0
289 Query	SET autocommit=0

// Named Lock 획득 시도
290 Query	select get_lock('1', 3000)
289 Query	select get_lock('1', 3000)

// Named Lock을 획득한 요청 수행
291 Query	SET autocommit=0
291 Query	select c1_0.coupon_id,c1_0.quantity from coupon c1_0 where c1_0.coupon_id=1
291 Query	insert into user_coupon (coupon_id) values (1) // 사용자 쿠폰 발급
291 Query	update coupon set quantity=1 where coupon_id=1  // 쿠폰 재고 감소
291 Query	commit
291 Query	SET autocommit=1
290 Query	select release_lock('1') // namedLock 반환
291 Query	SET autocommit=0
290 Query	commit

// Named Lock을 획득한 요청 수행
291 Query	select c1_0.coupon_id,c1_0.quantity from coupon c1_0 where c1_0.coupon_id=1
290 Query	SET autocommit=1
291 Query	insert into user_coupon (coupon_id) values (1) // 사용자 쿠폰 발급
291 Query	update coupon set quantity=0 where coupon_id=1 // 쿠폰 재고 감소
291 Query	commit
291 Query	SET autocommit=1
289 Query	select release_lock('1') // namedLock 반환
289 Query	commit
289 Query	SET autocommit=1
``````

위의 쿼리에서 확인 할 수 있듯이, `Named Lock`을 획득후에 쿠폰 발급 요청을 수행하는 것을 볼 수 있습니다.

## 쿠폰 발급에 적합한 동시성 제어 방법 선택

쿠폰 발급 요청은 짧은 시간에, 많은 요청이 들어오는 요청으로 예상할 수 있습니다.
따라서 쿠폰 발급 요청이 실패할 수 있는 `Optimistic Lock`을 이용한 동시성 제어는 사용하지 않는 것이 타당해 보입니다.

실제로 다음과 같이 각 동시성 방법에 따라, 요청 응답 시간이 달라지는 것을 확인할 수 있습니다.


> 실험 환경  

- 요청 쓰레드 : 32
- 쿠폰 발급 요청 개수 : 100_000
- datasource 커낵션 풀 : 40

|동시성 제어 방법| 요청 처리 시간(ms)|
|:--:|:--:|
|synchronized|109286|
|Pessimistic Lock|59861|
|Optimistic Lock|123885|
|Named Lock|88780|

위의 결과를 봤을 때, `Pessimistic Lock`을 이용한 동시성 제어 방법이 가장 빠른 것을 확인할 수 있습니다.

현재 `Named Lock`은 Mysql의 기능을 활용하여 구현하였기 때문에, `redis`를 이용한 `Named Lock`을 사용하면 위의 실험결과가 달라질 수 있습니다.
하지만 현재 개발중인 프로젝트에서는 redis를 사용하지 않기 때문에 `Pessimistic Lock`을 이용하여 동시성 문제를 해결하려 합니다.

끝까지 봐주셔서 감사합니다!
