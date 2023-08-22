---
layout: single
title: static 메서드가 Override 되지 않는 이유
categories: Java
tag: [Java, Collection]
toc: true
author_profile: false
search: true
sidebar:
    nav: "counts"
---
---
title:  "static 메서드가 Override 되지 않는 이유"

categories:
  - Java
tags:
  - [Java, Collection]

toc: true
toc_sticky: true

date: 2023-08-19
last_modified_at: 2023-08-19
---
# static 메서드가 Override 되지 않는 이유

Super class에 정의된 static 메서드는 Sub class 에서 Override될 수 없다.

Override될 수 없는 이유를 알기위해서는 먼저 Override가 어떻게 되는 지를 알아야 할 것 같다.

## Static Binding과 Dynamic Binding

Binding은 메서드의 호출 부분을 메서드의 정의 부분에 연결시켜 주는 것을 말한다.

Binding에는 static biding과 dynamic binding이 있다.

1. static Binding 
   - `컴파일 시점`에 객체의 타입이 결정되는 것을 static binding이라고 한다.
   - 컴파일러가 메서드의 이름에 메서드의 최종 구현을 맵핑해주는 것을 의미한다.
   - private, fianl, static method일 때 static binding을 한다.
2. Dynamic Binding
   - `런타임 시점`에 타입이 결정되는 것을 Dynamic Binding이라고 한다.

![](https://techvidvan.com/tutorials/wp-content/uploads/sites/2/2020/04/java-static-vs-dynamic-binding.jpg)

### Static Binding을 하는 이유

static binding은 compile-time에 실행된다.  따라서 런타임 시점에 method binding을 할 필요가 없기 때문에, 빠르게 실행될 수 있다.

> 출처  
> https://techvidvan.com/tutorials/static-and-dynamic-binding-in-java-differences-and-examples/  
> https://www.javatpoint.com/static-binding-and-dynamic-binding  
> https://www.codingninjas.com/studio/library/overloading-and-overriding-static-methods-in-java