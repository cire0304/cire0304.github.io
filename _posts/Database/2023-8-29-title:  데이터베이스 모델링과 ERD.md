---
title:  "데이터베이스 모델링과 ERD"

categories:
  - Database
tags:
  - Database

toc: true
toc_sticky: true

date: 2023-07-18
last_modified_at: 2023-07-18
---

본 글에서는 논리적 설계 단계에서 작성되는 ERD (Entity - Relationship Diagram)에 대해서 학습하려 합니다.

ERD(Entity-Relationship Diagram)는 데이터베이스 설계 과정에서 사용되는 개념적 도구로, 데이터베이스에 저장되어야 하는 데이터와 그들 사이의 관계를 시각적으로 표현합니다. ERD는 `개체(Entity)`, `속성(Attribute)`, 그리고 `관계(Relationship)`의 세 가지 주요 구성 요소로 이루어져 있습니다.

ERD를 작성하려면 먼저 애플리케이션의 요구사항을 분석해야 합니다. 요구사항 분석을 통해 데이터베이스가 처리해야 할 `엔터티`, `속성`, `관계` 등을 파악할 수 있습니다.

요구사항은 다음과 같습니다.

## 요구사항

### 회사 분석

-   회사는 부서로 구성되어 있다.
-   부서는 부서이름과 부서번호, 한 명의 부장을 가진다.
-   부장의 업무 시작날짜를 기록한다.
-   부서는 하나 이상의 장소에 있을 수 있다.

### 프로젝트

-   부서는 여러 프로젝트를 관리한다.
-   프로젝트는 고유번호, 고유이름, 위치 정보를 가진다.
-   한 프로젝트는 하나의 부서에만 속한다.

### 사원

-   사원은 한 부서에 속한다.
-   사원은 하나 이상의 프로젝트에 참여한다.
-   사원 데이터는 사번, 이름, 주소, 성별, 주소, 연봉 정보등이다.
-   각 사원은 프로젝트에 참여한 시간을 관리한다.
-   각 사원은 1 또는 0명의 직속 상관이 있다.

### 사원가족

-   사원들의 경조사 등을 위해 가족 정보를 유지한다.
-   가족 구성원에 대해 이름, 성별, 생일, 관계만 저장한다.

## 개념적 설계 : ERD

위의 요구 사항을 그림으로 그리면 다음과 같습니다.

![](https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2FcUr1O4%2FbtscCCHvkPB%2FE93YDMFK3rXo3GeFkcZGVk%2Fimg.png)

### 개체와 속성 (Entiy and Attribute)

> 개체

-   데이터베이스에서 표현하려는 개체나 정보의 집합으로, 일반적으로 테이블로 구현됩니다.

> 속성

-   엔티티의 세부 정보를 나타내는 구성 요소로, 테이블의 열(column)에 해당합니다.
-   키 속성 : 개체마다 고유한 값을 가지는 속성

![](https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2FbJGqZP%2FbtscGTvaGFZ%2Fkd0dZx3giN6Tv2W1HkRmpK%2Fimg.png)

---

### 관계 (Relationship)

관계는 서로 다른 개체간의 연결을 나타내며, 1:1, 1:N, M:N와 같은 다양한 유형의 관계를 표현할 수 있습니다.

관계는 선으로 표현되며, 선은 한개로 표현될 수도, 2개로 표현될 수도있습니다.
만약 사원이 부서에서 `반드시` 일해야만 한다면, `2개의 선`으로 관계를 표현합니다.
이러한 관계를 `total or mandatory participation` 이라고 말합니다.

하지만 사원(부장)이 부서를 관리할 수도, 그렇지 않을 수도 있습니다. 이를 1개의 선으로 관계를 표현합니다. 이러한 관계를 `partial or optional participation`  이라고 말합니다.

추가적으로 관계도 속성을 가질 수 있습니다.

![](https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2FcEIlo4%2FbtscKFPPTdr%2FDsY44TRUKruisVG8tKxv40%2Fimg.png)


### 약개체(weak entity)와 식별관계(identifying relationship)

> 약개체

다른 개체에 존재를 의존하고 있는 개체를 약개체라고 합니다.
여기서 존재를 의존하고 있다는 의미는, 의존의 대상이 되는 개체가 없다면 약개체 또한 없다는 의미입니다. 예를 들면, 사원(EMPLOYEE) 개체가 없다면 그에 딸린 가족(DEPENDENT) 개체또한 존재할 수 없습니다.

약개체는 부분 키(Partial Key)를 1개 이상 가질 수 있으며, 이를 통해 의존 대상인 개체와 함께 고유한 식별이 가능해집니다. 이렇게 부분 키와 의존 대상인 개체의 기본 키를 함께 사용하여 약개체를 고유하게 식별할 수 있습니다.

부분 키와 기본 키는 다릅니다. 부분 키는 약개체를 식별하기 위해 존재하며, 의존 하고 있는 개체의 외래 키와 함께 사용이 됩니다.  
{: .notice--info}

> 식별관계

식별 관계는 약개체와 그것의 부모 개체 사이의 관계를 의미합니다.

![](https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2FytuIt%2FbtscGI1mFm3%2FvIKx3hssHrKbZegGPosqkK%2Fimg.png)


### 다중값 속성(Multivalued Attribute)와 파생 속성(Derived Attribute)

> 다중값 속성

다중값 속성은 하나의 개체가 여러 개의 값을 가질 수 있는 속성입니다. 예를 들어, 한 사람이 여러 개의 전화번호를 가질 수 있는 경우 전화번호 속성은 다중값 속성이 됩니다.

> 파생 속성

파생 속성은 다른 속성들로부터 계산되거나 추론되는 속성입니다. 이러한 속성은 데이터베이스에 직접 저장되지 않을 수도 있으며, 필요할 때 계산되어 사용됩니다.

![](https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2F1AGuJ%2FbtscQxywxzt%2Fgc6qZuvwRFIZgLwhE6fvQk%2Fimg.png)

위와 같이 ERD를 사용하면 데이터베이스의 구조를 이해하기 쉽게 시각화할 수 있으며, 다양한 구성 요소와 관계를 정확하게 정의하고, 설계 오류를 줄여 데이터베이스를 효율적으로 구축할 수 있습니다.

## 기호 요약

![](https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2FdEBDRT%2FbtscHmRlDl8%2FMKRyhMdGTRrzqkm5KfpD21%2Fimg.jpg)

<br>
<br>

> 출처  
> [https://www.slideshare.net/hoyoung2jung/ss-40851049](https://www.slideshare.net/hoyoung2jung/ss-40851049)  
> [https://medium.com/omarelgabrys-blog/database-modeling-entity-relationship-diagram-part-5-352c5a8859e5](https://medium.com/omarelgabrys-blog/database-modeling-entity-relationship-diagram-part-5-352c5a8859e5)