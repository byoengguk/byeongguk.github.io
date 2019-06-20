---
layout: post
title: 블로킹에서 리액티브 까지
permalink: /to-reactive-from-blocking
categories: [it]
tags: [reactive]
---

Reactive
========

# 포스트 계기

이곳저곳에서 리액티브 이야기가 많이 들려온다.
사실, 여기에 관심을 갖게 된 것은 spring-webflux 때문이다.

spring-webmvc 로 빅히트를 친 전적이 있는, spring-framework 에서 새로 내놓은 웹 개발 프레임워크.
webmvc 놔두고, 왜 webflux 를 만들었고, 왜 또 그것을 그렇게 밀고 있을까?

과연 내 코딩 라이프에 webflux 가 webmvc 를 대체하게 될까?
지금 생각해보면, 이런 의구심으로 이 리액티브 세계에 발을 들이밀기 시작한 것 같다.

# 포스트 목표

* 리액티브가 뭐야? 라고 했을 때 대답할 수 있기.
* 그게 왜 좋아? 에 대답할 수 있기.

# Intro

[Node JS](https://nodejs.org/ko/about/) 소개 페이지에 아래와 같은 문구가 있다.

> 이는 오늘날 OS 스레드가 일반적으로 사용하는 동시성 모델과는 대조적입니다. 스레드 기반의 네트워크는 상대적으로 비효율적이고 사용하기가 몹시 어렵습니다. 게다가 잠금이 없으므로 Node의 사용자는 프로세스의 교착상태에 대해서 걱정할 필요가 없습니다. Node에서 I/O를 직접 수행하는 함수는 거의 없으므로 프로세스는 결과 블로킹 되지 않습니다. 아무것도 블로킹 되지 않으므로 Node에서는 확장성 있는 시스템을 개발하는 게 아주 자연스럽습니다.
  
이게 도대체 무슨 말일까?

# 코드로 이해해보자

간단한 에코 네트워크 서버들을 통해서, 저 문구를 이해해보자.


## SingleServer

```java
final ServerSocket serverSocket = new ServerSocket(port);

while (!Thread.currentThread().isInterrupted()) {
    final Socket socket = serverSocket.accept(); // 연결 블로킹

    final InputStream is = socket.getInputStream();

    byte[] buffer = new byte[2048];

    while (true) {
        final int bytesRead = is.read(buffer); // 수신 블로킹

        if (bytesRead == -1) {
            break;
        }

        final OutputStream os = socket.getOutputStream();
        os.write(buffer, 0, bytesRead);
        os.flush();
    }
}
```
[View in github](https://github.com/thywhite/tcp-echo-servers/blob/master/src/main/java/org/thywhite/springio/sharing/SingleServer.java)

### 설명
* java network api 만 사용한, 단순무식 echo server
* 최초 서버 실행시, ```serverSocket.accept()``` 에서 블로킹이 걸린다.
* 클라이언트 연결 시, 블로킹이 풀리며 계속 실행된다.
* 이후 데이터 수신 시,  ```is.read(buffer)``` 에서 블로킹이 걸린다.
* 클라이언트가 적절한 데이터를 보내고 나면, 블로킹이 풀린 후 받은 데이터를 다시 클라이언트에 보낸다.
* 그리고 데이터를 다시 기다린다.

### 문제점
* 서버가 여러 클라이언트를 상대할 수가 없다.
* 한 클라이언트를 상대할 때, ```is.read(buffer)``` 에서 블로킹 되어 있으므로, ```serverSocket.accpet()``` 를 실행할 수 없기 때문

* ![테스트 GIF](/img/tcp-echo-server/SingleServer.gif)
* [테스트 동영상](https://www.youtube.com/watch?v=9eatM-FHQCc&feature=youtu.be)

* 두 번째 연결은 펜딩 되어 있는 상태로, 첫 번째 연결이 끝난 후에야 에코 응답을 받는 것을 확인할 수 있다. 

* 실제로 이런 서버를 만드는 일은 없을 것이다.

## ThreadServer

```java
ExecutorService threadPool = Executors.newFixedThreadPool(2);
ServerSocket serverSocket = new ServerSocket(port);

while (!Thread.currentThread().isInterrupted()) {
    final Socket socket = serverSocket.accept();
    threadPool.submit(() -> { // 클라이어늩 데이터 수신을 thread pool에 맡긴다. 
        final InputStream is = socket.getInputStream();

        byte[] buffer = new byte[2048];

        while (true) {
            final int bytesRead = is.read(buffer);

            if (bytesRead == -1) {
                break;
            }

            final OutputStream os = socket.getOutputStream();
            os.write(buffer, 0, bytesRead);
            os.flush();
        }
        return null;
    });
}

threadPool.shutdownNow();
```
[View in github](https://github.com/thywhite/tcp-echo-servers/blob/master/src/main/java/org/thywhite/springio/sharing/ThreadServer.java)

### 설명
* SingleServer 에서 클라이언트를 상대하는 일을, 별도 쓰레드에 맡기는 방식
* ```serverSocket.accept()``` 만 하는 쓰레드 하나, 클라이언트 별로 ```is.read(buffer)```를 하는 쓰레드 들로 구성된다.
* ```is.read(buffer)``` 에서 블로킹이 걸리는건 똑같지만, 쓰레드 풀에 있는 쓰레드에서 걸리므로, 메인 쓰레드는 계속 ```serverSocket.accept()``` 를 수행할 수 있다.
* 동시 접속 가능 수는, 쓰레드 풀에 있는 쓰레드 숫자와 동일하다.
* 예제에서는 2로, 클라이언트를 2개나 상대 가능하다.(이전 서버 대비 100% 향상이다.)

* 톰캣(bio connector)및 쓰레드 풀을 기반으로 하는 WAS 들도, 이런 구조로 되어 있다고 보면 된다. 

### 쓰레드 수를 늘려보면?
* 쓰레드 수를 늘릴 수록 동시 접속 가능 수가 늘어난다.
* 그렇다고, 너무 많은 쓰레드를 사용하게 되어버리면, 쓰레드 간 실행 순서 조정 및 실행 상태 변경에 들어가는 비용이 커져버린다.
  * 전문용어로는 컨텍스트 스위칭 비용이 커진다고 한다.
* 바로 이 점이, Node JS 에서 쓰레드 기반 네트워크가 비효율적이라고 깐 부분이다.

