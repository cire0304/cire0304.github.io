---
title:  "Charset과 인코딩"

categories:
  - Web-etc
tags:
  - [Web, Encoding]

toc: true
toc_sticky: true

date: 2023-08-29
last_modified_at: 2023-08-29
---

HTML 문서를 보다보면 상단에 꼭 들어가는 태그가 있습니다. 바로 `charset`와 `UTF-8` 이라는 문자가 바로 이것입니다.

```html
<!doctype html>
  <html>
    ...
    <head><meta charset="UTF-8">
    ...
  </html>
```

`charset`과 `UTF-8`은 무엇이고 왜 들어가는 걸까요? 없다면 어떻게 되는걸까요? 궁금해져서 조사를 해보았습니다.

## charset

`charset`에 대해서 알기위해서는 먼저 문자열 `인코딩`의 개념과 `디코딩`의 개념을 알아야합니다.  

컴퓨터는 문자라는 것을 알지못합니다. 그저 \`0\`과\`1\` `바이너리 데이터`로 모든 것을 처리합니다. 따라서 문자를 바이너리 데이터로 *변환하는 과정*을 `인코딩`, 반대의 경우를 `디코딩`이라고 합니다. 그리고 인코딩과 디코딩을 하기 위해 필요한 것이 `charset(문자표)`입니다. 이러한 charset은 여러가지가 있고, 대표적인 charset으로는  `ASCII 코드`가 있습니다.

![](https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2FqOPNt%2FbtrAdcY26CF%2FKsn1qKzUqEaCql1Cbk6GG0%2Fimg.png)

위의 ASCII 코드 `인코딩`이라는 과정을 거쳐 문자를 숫자(바이너리 데이터)로 표현이 가능해집니다.  
예를 들어 ASCII 코드 이용해서 \`a\`(문자)를 \`0X61\`라는 숫자(바이너리 데이터)로 변환할 수 있습니다.

{: .notice--info}  
ASCII 코드에 영문자와 특수문자만 포함되어 있는 것은, 미국에서 제작되었고 다른 문자(한글, 한자, 일본어)를 표현하는 것을 고려하지 않아서 그렇습니다.

## ASCII 코드의 한계점과 UNICODE

위의 ASCII 코드을 보면 *영문자와 특수문자만* 포함되어 있는 것을 알 수 있습니다. 하지만 전세계에서 인터넷을 사용하고 있고, 따라서 각 나라마다 사용하는 문자 또한 바이너리 데이터로 변환해야할 필요성이 나타나기 시작하였습니다. 그 결과 다양한 `charset`이 만들어지게 되었고, 그 중 하나가 `UNICODE`입니다. UNICODE는 전 세계의 모든 문자를 다룰 수 있도록 만들어진 문자 코드입니다.  

## UTF-8 : 인코딩 방식 중 하나

다양한 `charset`이 있듯이 다양한 `인코딩`방식이 있습니다. 그 중 하나가 `UTF-8`입니다. UTF-8은 ASCII 코드와 UNICODE를 사용한 인코딩으로, ASCII 코드표로 변환가능하면 ASCII 코드로 변환하고, 그 이외의 문자에 대해서는 UNICODE로 변환합니다.

{: .notice--info}  
UTF-8 방식의 인코딩외에 한글을 바이너리 데이터로 표현하기 위한 EUC-KR 방식의 인코딩도 있습니다.

> URL 인코딩

URL 인코딩이란 URL에서 URL로 사용할 수 없는 문자 혹은 URL로 사용할 수 있지만 의미가 왜곡될 수 있는 문자들을 '%XX'의 형태로 변환하는 것을 말합니다. 

웹사이트의 주소 뒤에 URL 파라미터를 넣어서 서버에 인자를 보낼 수 있습니다. 이때, *URL의 문자는 오로지 ASCII 문자여야하기 때문에* ASCII 문자가 아닌 문자는 *UTF-8 인코딩을 사용하여 변환*해줘야 합니다.

또한, URL에서는 공백 문자가 허용되지 않기 때문에 공백 문자는 '%20' 혹은 '+'로 인코딩이 됩니다.

{: .notice--info}  
자세한 내용은 [RFC2396](https://datatracker.ietf.org/doc/html/rfc2396)을 참고해주세요.

## 정리

결국 인코딩이라는 것은 ASCII 코드와 ASCII 코드로 표현되지않는 문자(한글, 한자, 일본어)에 대해 바이너리로 표현하는 규칙입니다. HTML 헤더에 `charset=UTF-8` 메타데이터가 존재하는 이유도, HTML 문서의 문자열 데이터가 어떤 인코딩을 사용하여 변환됬는지를 나타내기 위함이였습니다. `EUC-KR`로 인코딩된 한글 문자열을 `UTF-8` 로 디코딩할 경우 깨지는 이유는 인코딩과 디코딩시 사용했던 바이너리 변환규칙이 달랐기 때문입니다.

<br>
<br>

> 출처  
> [IT 엘도라도](https://it-eldorado.tistory.com/143)  
> [위키백과](https://ko.wikipedia.org/wiki/UTF-8)  
> [널널한 개발자](https://www.youtube.com/watch?v=6hvJr0-adtg&ab_channel=%EB%84%90%EB%84%90%ED%95%9C%EA%B0%9C%EB%B0%9C%EC%9E%90TV)
