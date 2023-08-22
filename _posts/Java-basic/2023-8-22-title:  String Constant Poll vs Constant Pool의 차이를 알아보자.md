---
title: "String Constant Poll vs Constant Pool의 차이를 알아보자"

categories:
  - Java-basic
tags:
  - Java

toc: true
toc_sticky: true

date: 2023-08-22
last_modified_at: 2023-08-22
---

# String Constant Poll vs Constant Pool

String Constant Poll과 Constant Poll은 이름이 비슷하다.
하지만 위의 두 용어는 완전히 다른 용어이다.

저장되는 위치, 저장하는 데이터의 종류, 관리 주체까지 모든 것이 다른 저장공간이다.

![](https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2FeaJffC%2FbtsgwMqYWOP%2FlWjjsBoXdDoUMWgrSr5ZCK%2Fimg.png)


|| String Constant Pool | Constant Pool inside a Java class file |	Runtime Constant pool |
|:-:|:-:|:-:|:-:|
|역할|	String 객체의 상수 값을 저장(캐싱)|클래스 파일의 상수 값을 저장|Class Constant pool에서 읽어온 상수 값, 클래스 메타데이터 저장|
|저장되는 값 | String 상수 값 <br> ex) "Hello", "World" | 클래스 파일에 포함된 상수 값 <br> ex) 123, 3.14, true | 클래스 파일에 포함된 상수 값. Class Constant pool에 저장되어있던 값이 런타임시 이 영역으로 저장된다. |
| 저장 위치 | Java 7 이전 - Perm <br> Java 8 이상 - Metaspace| Class file | 		Java 7 이전 - Perm <br> Java 8 이상 - Metaspace |
|저장 트리거| String.intern(), String을 리터럴로 생성 | 컴파일시 생성됨 | 클래스파일에 코드 레벨로 선언된 상수풀이 런타임시 로더의 판단에 의해 올라옴. |
|불변 여부|불변 여부	불변 (Immutable)|Class 파일 자체는 불변.|Runtime시 동적 로드에 의해 변경될 수 있음	불변이 아님.클래스파일이 동적으로 로딩되고 초기화되기 때문에, 로드될 때 마다 변경됨|

## String Constant Pool은 GC의 대상이 될까?

GC의 대상은 Heap 영역이라고 배웠다. 그리고 String Constant Pool이 저장되는 곳은 Metaspace(Java8 이상) 이라고 한다.

그렇다면 String Constant Pool에 저장되 있는 데이터는 GC의 대상이 되지 않는 걸까?

정답은 그렇다. String Constant Pool에 저장되어 있는 String literals은 GC의 대상이 되지 않는다.

다만 여기서 확실히 알아야 할 점은, 모든 String 객체가 String Constant Pool에 저장되는 것이 아니라는 점이다.
-> 동적으로 생성된 String 객체는 Heap 영역에 저장된다.


> 출처  
> https://deveric.tistory.com/123
> https://stackoverflow.com/questions/18406703/when-will-a-string-be-garbage-collected-in-java
