---
title:  "[토이 프로젝트] 초기 응답 지현 이슈 해결 : warm up"

categories:
  - Spring
tags:
  - [Spring, Java, JPA]

toc: true
toc_sticky: true

date: 2023-10-19
---


# 들어가며

쿠폰을 발급하고, 사용할 수 있는 토이 프로젝트를 진행하던 중, 배포 시점에 쿠폰 조회 요청에 대한 응답이 매우 늦어지는 현상을 발견했습니다.

![](../../image/Toy-Project/[토이%20프로젝트]%20초기%20응답%20지현%20이슈%20해결%20:%20warm%20up/쿠폰%20조회%20초기%20응답%20지연%20형상%20-%201.png)

초기 요청에 대한 최대 응답 시간은 11초 정도로 매우 느리다는 것을 확인할 수 있었습니다.

하지만 위의 그림을 보면 지연 현상이 계속 유지되는 것이 아니라 어느 정도 시간이 지나면 응답 지연이 해소된다는 점이 특징입니다.
이번 포스팅은 이러한 현상이 왜 일어나고, 어떻게 해결했는지에 대해 설명하려합니다.

# 왜 배포 시점에만 응답이 느릴까?

초기 응답 지연 현상의 원인은 크게 두 가지로 추측해볼 수 있습니다.

## 클래스 로더의 Lazy loading

JVM은 자바의 클래스를 읽어오기 위해서 클래스 로더를 사용합니다.  
클래스 로더는 사용하여 클래스 파일을 찾고, 메모리에 로드해 실행 가능한 상태로 만드는 역할을 합니다.

그리고 클래스 로더는 `Lazy loading` 방식으로 동작합니다. `Lazy loading`은 애플리케이션이 시작될 때 로딩되는 것이 아니라, 클래스가 필요한 시점까지 로딩을 지연하는 방식입니다.  

배포 시점에는 대부분의 클래스들이 한번도 사용되지 않았기 때문에, 클래스 로더에 의해 메모리에 적재되지 않은 상태입니다.  
그런 상태에서 웹 애플리케이션 요청이 들어오게 되면, 그때 클래스 로더의 `Lazy loading`에 의해 클래스들이 메모리에 적재됩니다.

따라서 `Lazy loading`에 의해 응답 지연이 발생합니다.

## JIT 컴파일러


자바는 소스 코드를 바이트 코드로 컴파일하고, 바이트 코드를 JVM에서 실행하는 하이브리드 언어입니다.  
그리고 JVM은 바이트 코드를 한줄 한줄 읽어, 기계어로 번역한 뒤 실행합니다. 이러한 과정을 인터프리트라고 합니다.

하지만 위의 과정은 다른 컴파일 언어에 비해서 **비교적 실행 속도가 느리다**라는 단점이 있는데요. 왜냐하면, 다른 c 언어와 같은 컴파일 언어는 소스코드를 컴파일 할 때, **코드 최적화**를 수행하기 때문입니다. 따라서 이미 최적화가되어 있는 준비되어 있는 기계어를 읽는 컴파일 언어에 비해서 인터프리터 언어는 성능이 부족할 수 밖에 없습니다.

자바는 위에서 말했던 성능 부족 문제를 해결하기 위해 Java 1.3 HotSpot VM에서 부터 바이트 코드를 적시에 기계어를 만들어내는 JIT(Just In Time) Compiler를 도입하여, 빠르게 기계어를 실행할 수 있도록 하였습니다.

JIT 컴파일러는 바이트 코드를 기계어로 변환하는 과정에서, 기계어를 캐시에 저장하고 활용하게 됩니다.  
이를 통해 반복되는 기계어 변환 과정을 줄이게 되어 성능을 향상시키게 되고, 런타임 환경에 맞춰 코드를 최적화 함으로써 추가적인 성능향상을 이루게 됩니다.


## JIT Tiered Compilation

JIT 컴파일러는 얼마나 많이 메서드의 호출했냐에 따라서, 얼만큼 컴파일 최적화를 할지 결정합니다. 

- Level 0 - Interpreted Code: JVM은 초기에 모든 코드를 인터프리터를 통해 실행한다. 이 단계는 앞서 살펴본것과 같이, 컴파일된 기계어를 실행하는 것보다 성능이 낮다.
- Level 1 - Simple C1 Compiled Code: Level 1 은 JIT 컴파일러가 단순하다고 판단한 메서드에 대해 사용된다. 여기서 컴파일된 메서드들은 복잡도가 낮아, C2 컴파일러로 - 컴파일한다고 하더라도 성능이 향상되지 않는다. 따라서 추가적인 최적화가 필요 없으므로 프로파일링 정보도 수집하지 않는다.
- Level 2 - Limited C1 Compiled Code: 제한된 수준으로 프로파일링과 최적화를 진행하는 단계이다. C2 컴파일러 큐가 꽉 찬경우 실행된다.
- Level 3 - Full C1 Compiled Code: 최대 수준으로 프로파일링과 최적화를 진행한다. 즉 일반적인 상황에서 수행된다.
- Level 4 - C2 Compiled Code: 애플리케이션의 장기적인 성능을 위해 C2 컴파일러가 최적화를 수행한다. Level 4에서 최적화된 코드는 완전히 최적화 되었다고 간주되어, 더이상 프로파일링 정보를 수집하지 않는다.

서버가 배포된 직후에는 JIT 컴파일러는 아무런 코드도 기계어로 컴파일 하지 않았으며, 따라서 코드 캐시에 적재된 기계어도 존재하지 않습니다.
따라서 배포 직후 시점의 코드는 인터프리터에서 실행되거나, C1 혹은 C2 컴파일러가 최적화하고 컴파일 과정이 동반되므로 필연적으로 성능 저하가 발생할 수 밖에 없습니다.

그러므로 `JTI Compile`에 의해 응답 지연이 발생합니다.

출처 : https://hudi.blog/jvm-warm-up/
{: .notice--info}


## 어느 시점에 JIT 컴파일러가 동작할까?

아래의 명령어를 통해 JIT 컴파일러가 언제 동작하는 알 수 있습니다.
```text
$ java -XX:+PrintFlagsFinal -version | grep Threshold | grep Tier
```

```text
openjdk version "17.0.8" 2023-07-18
OpenJDK Runtime Environment JBR-17.0.8+7-1000.22-nomod (build 17.0.8+7-b1000.22)
OpenJDK 64-Bit Server VM JBR-17.0.8+7-1000.22-nomod (build 17.0.8+7-b1000.22, mixed mode)
    uintx IncreaseFirstTierCompileThresholdAt      = 50                                        
     intx Tier2BackEdgeThreshold                   = 0                                         
     intx Tier2CompileThreshold                    = 0                                         
     intx Tier3BackEdgeThreshold                   = 60000                                   
     intx Tier3CompileThreshold                    = 2000                                      
     intx Tier3InvocationThreshold                 = 200                                       
     intx Tier3MinInvocationThreshold              = 100                                       
     intx Tier4BackEdgeThreshold                   = 40000                                     
     intx Tier4CompileThreshold                    = 15000                                     
     intx Tier4InvocationThreshold                 = 5000                                      
     intx Tier4MinInvocationThreshold              = 600                                       
```

- InvocationThreshold : 메서드의 호출 수
- BackEdgeThreshold : 하나의 메서드 내의 반복문 횟수
- CompileThreshold : InvocationThreshold + BackEdgeThreshold 

# warm up 이란?

위의 내용을 통해 응답 지연이 발생하는 원인은 두 가지로 볼 수 있습니다.
- 클래스는 필요할 때, Lazy loding으로 메모리에 적재된다.
- JVM은 자주 실행되는 실행되는 코드를 컴파일하고 캐시한다.

그렇다면 초기 응답 지연을 막기위해 배포 하기 전에 미리 클래스를 메모리에 적재하고, 많이 사용할 것 같은 메서드가 기계어로 캐시될 수 있도록 하면 어떨까요?  
그러한 과정을 `warm up`이라고 합니다.


## 어떠한 메서드가 응답 지연에 영향을 줬을까?

`warm up`을 진행하기 위해서는 어떤 메서드가 많이 호출되었는지, 또 어느 메소드에서 시간이 많이 걸렸는지 확인이 필요합니다.
저는 Pinpoint를 사용하여 메서드 실행 시간을 측정하였습니다.

![](../../image/Toy-Project/[토이%20프로젝트]%20초기%20응답%20지현%20이슈%20해결%20:%20warm%20up/쿠폰%20조회%20초기%20응답%20지연%20현상%20-%202.png)

- doGet(...) : 3.078초
- inovokeWithinTransaction(...) : 6.317초

측정 결과를 통해, 메서드의 실행속도가 매우 느려  `warm up` 과정이 필요함을 알 수 있었습니다.

## Warm up 코드

`warm up`을 하는 방법은 다양하지만, 저는 이벤트 기반으로 코드를 작성하였습니다.

```java
@Component
@RequiredArgsConstructor
public class CouponWarmupRunner {

    private final static Logger log = LoggerFactory.getLogger(CouponWarmupRunner.class);
    private static final RestTemplate restTemplate = new RestTemplate();
    private static final Integer WARM_UP_COUNT = 코드 실행 횟수
    private static final String COUPONS_URL = API URL


    @EventListener(ApplicationReadyEvent.class)
    public void warmup() {
        log.info("start Coupon warm up");
        try {
            IntStream.rangeClosed(1, WARM_UP_COUNT).forEach(i -> {
                restTemplate.getForObject(COUPONS_URL, String.class);
            });
        } catch (Exception e) {
            log.warn("CouponWarmupRunner caught exception at inner. ex {}", e.getMessage());
        }
        log.info("finish Coupon warm up");
    }

}
```

## JIT 로그를 확인하는 방법

아래와 같이 JVM 옵션을 추가하면 **hotspot_pid{number}.log** 파일을 생성합니다. 이 로그를 통해 JIT 컴파일 관련 로그를 확인 할 수 있습니다.
```
VM Options
-XX:+UnlockDiagnosticVMOptions -XX:+LogCompilation
```

아래의 로그를 통해 invokeWithinTransaction(...) 메서드가 level3 으로 컴파일 된 것을 확인할 수 있습니다.
```
<task_queued compile_id='11625' method='org.springframework.transaction.interceptor.TransactionAspectSupport invokeWithinTransaction (중간 로그 생략) bytes='521' count='1792' iicount='1792' level='3' stamp='20.640' comment='tiered' hot_count='1792'/>

<nmethod compile_id='11625' compiler='c1' level='3' (중간 로그 생략) count='1792' iicount='1792' stamp='20.648'/>

<task compile_id='11625' method='org.springframework.transaction.interceptor.TransactionAspectSupport invokeWithinTransaction (중간 로그 생략) bytes='521' count='1792' iicount='1792' level='3' stamp='20.645'>
```

## warm up 진행 후 배포 결과

![](../../image/Toy-Project/[토이%20프로젝트]%20초기%20응답%20지현%20이슈%20해결%20:%20warm%20up/쿠폰%20조회%20초기%20응답%20지연%20현상%20-%203.png)

`warm up`을 진행하게 되면, 웹 애플리케이션 서버에 실제 요청을 보내게 됩니다.  
그리고 `warm up` 이후 초기 응답 지연 현상이 사라진 것을 확인할 수 있습니다.

끝까지 봐주셔서 감사합니다!


> 참고  
> [[Backend] JVM warm up / if(kakao)dev2022](https://www.youtube.com/watch?v=utjn-cDSiog&ab_channel=%EC%B9%B4%EC%B9%B4%EC%98%A4)  
> [스프링 애플리케이션 배포 직후 발생하는 Latency의 원인과 이를 해결하기 위한 JVM Warm-up](https://hudi.blog/jvm-warm-up/)  
> [토비의 스프링 부트 1 - 스프링 부트 앱에 초기화 코드를 넣는 방법 3가지](https://www.youtube.com/watch?v=f017PD5BIEc&ab_channel=%ED%86%A0%EB%B9%84%EC%9D%98%EC%8A%A4%ED%94%84%EB%A7%81)  
