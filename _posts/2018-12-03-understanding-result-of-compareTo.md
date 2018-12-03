---
layout: post
title: compareTo return value 이해하기
permalink: /understanding-result-of-compareTo
categories: [code]
tags: [java]
---

처음으로 써 보는 코드 이야기
-------------------
맙소사, 세달만인다.
써보고 싶은건 많았는데, 내 행동력이 부족한건 생각도 못했다.
아니, 그냥 무시했다.
현실적으로 한달에 한번 정도는 써보도록 하자.

복잡한 이야기는 쓰기가 어려우니, 간단한 `compareTo` 결과 값에 대한 이야기나 주절주절 해보자.


int compareTo(Object o)
-------------------------
정렬과 관련된 메소드로, 간단하지만 구현하기는 어려운 메소드 이다.
(개인적으로는 Guava `comparatorChain`을 선호하지만, 취향껏 알아서 구현하자.)
여기서 이야기하고 싶은건, 이게 무슨 메소드고, 어떻게 구현하는게 좋은가에 대한 것은 아니다.

난 단지, 이 메소드의 리턴 타입, 저 primitive int 에 대한 것만 이야기 할 것이다.

만약 누군가 나에게 비교를 위한 메소드를 정의하라고 하면, 아래와 같이 정의 할 것이다.

```java
public interface Comparable {
    CompareResult compareTo(Object o);
}

enum CompareResult {
    THIS_IS_GRATER,
    EQUALS,
    OTHER_IS_GREATER
}
```

차이점은 return value를 Enum 으로 정의, 이렇게 하면 compareTo return value를 이해하기 쉬울 것이라고 생각했다.

(글 쓰면서 느낀 점인데, `Comparator.compare`에는 다른 Enum이 적용되어야겠다-_-;)

이렇게 정의하려고 생각한 이유는, int type의 return value 가 직관적으로 와닿지 않았기 때문이었다.

실제 java doc 에서는 아래와 같이 기재하고 있다.

> @return  a negative integer, zero, or a positive integer as this object is less than, equal to, or greater than the specified object.

그러니 저 결과 값을 사용하는 코드는 보통 아래와 같이 짜여지기 마련이였다.

```java
int compareTo = aObject.compareTo(bObject);

if (compareTo > 0) {
    doSomeThingWhenBIsGreater();
} else if (compareTo < 0) {
    doSomeThingWhenAIsGreater();
} else {
    doSomeWhenEquals();
}

```
이런 코드를 보면.. 보는 순간 바로 눈에 들어오지 않았다.
0 이면 같다는걸 알겠는데, 양수가 나오면 내가 큰건지 딴게 큰건지 헷갈렸다. 그래서 맨날 저 java doc을 다시 들여다보고, 아 딴게 큰거구나 하고 알아냈고, 바로 또 까먹고 나중에 또 찾아보게 되었다.

왜 return value로 객체간 순서를 정의했는지, 이해하지 못하고 한동안 시간을 보냈었다.
사실, compareTo 메소드 자체를 많이 쓸 일이 없기 때문에, 그냥 그려러니, 역사와 전통이 그런가 보다 하고 걍 지냈다.

그렇게 살다가..

# 갑자기 내 코드에 `BigDecimal` 이 많아지기 시작했다. 

도메인을 바꾸면서, 정확한 소수점 처리를 많이 해야되기 때문에, 저 `BigDecimal` 을 많이 쓰기 시작하게 됬다.
그런데 애는, `int`, `long`, `double` 등과 다르게, if 문 내에 `>`, `<=` 등을 쓸 수가 없다.

어쩔 수 없이, compareTo 메소드를 다시 쓰게 되었다.
여전히 return 값을 직접적으로 해석할 수는 없지만, 아래와 같은 놀라운 사실을 알게 되었다.

```java
BigDecimal a, b;
if (a.compareTo(b) < 0) {
    // doSomething when a is less than b  
}
```

위 문장을 `int` 로 바꾼다면..
```java
int a, b;
if (a < b) {
    // doSomething when a is less than b
}
```
차이가 느껴지는가?
그냥 compareTo와 0을 없애버리고, 있던 비교 연산자 compareTo 자리에 가져다가 넣으면 된다.

지금 생각해보면 compareTo를 꺼내썻던 것이, 이해를 더 어렵게 만든 것 같다.
위 팁을 깨닫고 난 뒤로는, 조금은 코드 읽기가 편해졌다. 만세~


Ending
--------
그나저나, 저 위에 것처럼 2번 쓸 때는 어떻게 해야하지? 고민 좀 더 해봐야겠다.
 
