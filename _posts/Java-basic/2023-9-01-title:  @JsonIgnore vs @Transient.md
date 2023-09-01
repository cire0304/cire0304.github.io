---
title:  "@JsonIgnore vs @Transient"

categories:
  - Java-basic
tags:
  - [Java, Serralization]

toc: true
toc_sticky: true

date: 2023-9-01
---

자바에는 객체를 바이너리 데이터로 직렬화 및 역직렬화할 수 있도록 `Serialization` 인터페이스를 지원하고 있습니다. 또한 특정 필드를 직렬화에서 제외시키는 `trasient` 키워드또한 존재합니다.

이러한 직렬화에서 제외 기능을 어노테이션으로도 사용이 가능합니다. 직렬화를 제외시키기 위한 어노테이션은 다양하게 있지만 그 중 `@JsonIgnore`와 `@Transient`에 대해서 알아보려 합니다.

{: .notice--info}  
엄밀히 말해서 @Transient은 직렬화에서 제외시키는 기능이 아닌, 영속화에서 제외시키는 기능입니다.

## @JsonIgnore

특정 메소드나 필드를 직렬화 과정에서 제외시키는 `@JsonIgnore` 어노테이션이 존재합니다. 이 어노테이션은 *Jackson* 라이브러리에 속한 어노테이션으로, 비밀번호와 같은 민감한 정보가 저장되지 않도록 하기위해 사용됩니다.

```java
class Person implements Serializable {

    private Long id;
    private String firstName;
    private String lastName;

    // getters and setters
}
```

위와 같은 클래스가 있고, Jackson 라이브러를 이용하여 Json 형식의 데이터로 변환하고 싶다고 가정하겠습니다.

```java
@Test
void initializeTest() throws JsonProcessingException {
    Person person = new Person(1L, "My First Name", "My Last Name");
    String result = new ObjectMapper().writeValueAsString(person);
    System.out.println(result);
    // 결과: {"id":1,"firstName":"My First Name","lastName":"My Last Name"}
}
```

하지만 Json으로 변환된 필드 중 `id`와 같은 필드는 저장(직렬화)되지 않길 원할 수 있습니다. 이때 사용되는 것이 `@JsonIgnore`입니다.

```java
class Person implements Serializable {

    @JsonIgnore
    private Long id;
    private String firstName;
    private String lastName;

    // getters and setters
}
```

```java
@Test
void initializeTest() throws JsonProcessingException {
    Person person = new Person(1L, "My First Name", "My Last Name");
    String result = new ObjectMapper().writeValueAsString(person);
    System.out.println(result);
    // 결과: {"firstName":"My First Name","lastName":"My Last Name"}
    // id 필드는 저장되지 않는다.
}
```

## @Trasient

@Trasient은 JPA의 영속화 과정에서 특정 필드를 제외시키기 위해 사용되는 어노테이션입니다. 필드에 @Transient를 붙이면, 더 이상 그 필드는 영속 대상이 아니며, 데이터베이스에 저장이 되지도 데이터베이스에서 값을 가져올 수도 없습니다.

```java
@Entity
@Table(name = "Users")
class User implements Serializable {

    @Id
    private Long id;
    private String username;
    private String password;
    @Transient
    private String repeatedPassword;

    // getters and setters
}
```

*여기서 주의해야할 점은 영속 대상에서 제외되는 것이지, 직렬화에서 제외되는 것이 아니라는 점입니다.

```java
@Test
void givenUser_whenSerializing_thenTransientFieldNotIgnored() throws JsonProcessingException {

    User user = new User(1L, "user", "newPassword123", "newPassword123");
    String result = new ObjectMapper().writeValueAsString(user);
    System.out.println(result);
    // 결과: {"id":1,"username":"user","password":"newPassword123","repeatedPassword":"newPassword123"}
    // repeatedPassword 필드는 직렬화 가능
}
```

## 정리

@JsonIgnore와 @Transient은 둘 다 특정 필드가 저장이 되지 않길 바랄 때 사용할 수 있습니다.  
@JsonIgnore은 Json 형식으로 데이터를 직렬화할 때 사용할 수 있고, @Trasient는 특정 필드가 영속 대상에서 제외되길 바랄 때 사용할 수 있습니다.

따라서 각각의 어노테이션을 목적에 맞게 잘 사용해야합니다.


> 출처  
> [https://www.baeldung.com/java-jsonignore-vs-transient](https://www.baeldung.com/java-jsonignore-vs-transient)
