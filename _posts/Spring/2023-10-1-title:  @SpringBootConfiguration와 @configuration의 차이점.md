---
title:  "@SpringBootConfiguration와 @configuration의 차이점"

categories:
  - Spring
tags:
  - [Spring, Java]

toc: true
toc_sticky: true

date: 2023-09-23
---

## 개요
스프링은 빈(객체)을 관리하는 컨테이너를 제공해주는 프레임워크입니다.
따라서 먼저 빈을 컨테이너에 먼저 등록을 해줄 필요성이 있습니다.

빈을 컨테이너에 등록하는 방법 중 `@SpringBootConfiguration`을 사용한 빈 등록과 `@configuration`을 사용한 빈 등록 방법은 무슨 차이점이 있을까요?

## @SpringBootConfiguration

먼저 `@SpringBootConfiguration`은 `@SpringBootApplication` 어노테이션에 선언되어 있습니다.

```java
// 코드 생략
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(...)
public @interface SpringBootApplication {
  // 코드 생략
}
```

`@SpringBootApplication` 내부 코드를 살펴 보면 크게 세 가지 중요 어노테이션이 선언되어 있는 걸 확인할 수 있었습니다.
- @SpringBootConfiguration
- @EnableAutoConfiguration
- @ComponentScan(...)

본 포스팅에서 주의 깊게 볼 어노테이션은 `@SpringBootConfiguration` 입니다.  

`@SpringBootConfiguration`을 사용하면 Spring Boot가 클래스 경로에 있는 라이브러리를 기반으로 다음과 같은 빈을 자동으로 등록합니다.

- Spring Boot가 제공하는 기본 빈
  - `ApplicationContext` 빈, `WebMvcConfigurationSupport` 빈
- Spring Boot Starter 라이브러리가 제공하는 빈
  - `WebServer` 빈, `DispatcherServlet` 빈
- 개발자가 @Bean 어노테이션을 사용하여 등록한 빈

## @Configuration

`@Configuration`은 `@Bean`과 함께 사용되며, 메서드로 빈을 등록해주기 위한 어노테이션 입니다.

`@ComponentScan`이 `@Configuration`이 붙은 클래스를 컨테이너에 빈으로 등록해주고, `@Bean` 어노테이션이 선언된 메서드로 반환되는 객체를 빈으로 등록해줍니다.

```java
@Configuration
public class ApplicationConfig {    
    @Bean
    public ArrayList<String> array(){
        return new ArrayList<String>();
    }   
}
```

## @SpringBootConfiguration vs @Configuration

`@SpringBootConfiguration`와 `@Configuration`의 주요 차이점은 `@SpringBootConfiguration`은 *구성 자동화*를 사용한다는 점입니다. *즉 Spring Boot 클래스 경로에 있는 라이브러리를 기반으로 자동으로 빈을 등록합니다.*

> @SpringBootConfiguration is an alternative to the @Configuration annotation. The main difference is that @SpringBootConfiguration allows configuration to be automatically located. This can be especially useful for unit or integration tests. - [Guide to @SpringBootConfiguration in Spring Boot](https://www.baeldung.com/springbootconfiguration-annotation)

따라서 `@SpringBootConfiguration`는 한 서비스 내에 한 개의 클래스에서만 선언이 되고, `@Configuration`은 여러 클래스에 선이 될 수 있습니다.

> 출처  
> [Guide to @SpringBootConfiguration in Spring Boot](https://www.baeldung.com/springbootconfiguration-annotation)
