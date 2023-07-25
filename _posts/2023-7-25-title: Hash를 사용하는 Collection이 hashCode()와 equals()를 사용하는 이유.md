---
layout: single
title: Hash를 사용하는 Collection이 hashCode()와 equals()를 사용하는 이유
---

## Hash Collection의 동등성 판단

HashSet이나 HashMap과 같은 hash를 사용하는 Collection들은 아래 그림과 같은 방법으로 객체가 동등한지 를 판단합니다.

![image](https://github.com/cire0304/cire0304.github.io/assets/80495427/1427000a-0038-4844-bd31-81b4d19acb0f)

> hashCode() : 객체의 주소 값을 이용해서 해싱 기법을 통해 만든 해시 코드 반환  
> equals() : 객체가 저장된 메모리 주소 값 반환

## equals()만으로 동등성 비교가 가능하지 않을까?

hash를 사용하는 Collection들은 왜 `hashCode()` 와 `equals()` 메서드를 함께 사용할까요?

`equals()` 메서드로 충분히 동등한지 아닌지 판단 할 수 있을거라고 생각했습니다.

분명 Collection 자료형을 만드시분들이 저렇게 만든 이유가 있을거라고 생각했습니다.

공부해보니 다음과 같은 이유가 있었습니다.

## hashCode()를 사용하는 이유

제가 생각하기에 `hashCode()`를 사용하는 이유는 Hash 알고리즘을 사용하기 때문이라고 생각합니다.

(적고 나니 정말 너무나 당연한 이유였네요…)

Hash 알고리즘은 Hash function을 이용해서 키값을 hash code로 변환 후 이 값을 버킷 값으로 이용하는 알고리즘 입니다. 

자바에서는 hashCode() 메서드 반환 값이 버킷 값으로 사용되는걸로 이해했습니다.

![image](https://github.com/cire0304/cire0304.github.io/assets/80495427/c9064d1d-1ad4-4368-8c23-5f1d819a1f7e)

(출처 : https://www.geeksforgeeks.org/implementing-our-own-hash-table-with-separate-chaining-in-java/)

위의 그림과 같이 Hashing 된 값(hash code)에 따라 적절한 인데스에 자료들을 저장함으로써, 빠르게 데이터를 찾을 수 있게 됩니다.

그리고 적절한 인데스를 찾았다면, 그 인데스에 저장되어 있는 1개 이상의 데이터들을 `equals()` 메서드를 사용하여 동등성을 비교하면서 데이터를 조회하거나 추가하는게 가능해집니다.

## 정리

`equals()` 메서드만 가지고 동등성을 비교할 수는 있었습니다.

하지만, hashCode()를 함께 사용함으로써 `더 빠른 검색 시간`을 제공할 수 있었던 것입니다.

HashMap은 기초적인 자료구조임에도 잊고있었네요….

기본부터 다시 공부할 필요성을 느꼈습니다 ㅜㅜ
