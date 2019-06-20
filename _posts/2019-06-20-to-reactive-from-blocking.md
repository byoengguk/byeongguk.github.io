---
layout: post
title: 블로킹에서 리액티브 까지
permalink: /to-reactive-from-blocking
categories: [it]
tags: [reactive, java, tcp, network]
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

### 테스트 영상
![테스트 GIF](/img/tcp-echo-server/single-server.gif)
* [View in youtube](https://youtu.be/9eatM-FHQCc)

* 두 번째 연결은 펜딩 되어 있는 상태(backlog 큐에 들어가 있다)로, 첫 번째 연결이 끝난 후에야 에코 응답을 받는 것을 확인할 수 있다. 

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
* 실제로 저 에코 서버는 하는 일이 없지만, 저 모델로 만들게 되면 수만 동접은 무리이다.

### 테스트 영상
![테스트 GIF](/img/tcp-echo-server/thread-server.gif)
* [View in youtube](https://youtu.be/NdGKleVINI0)

* 4 세션을 연결했지만, 동시에 최대 2세션까지만 에코 응답이 오는 것을 알 수 있다. 

## Non blocking server

```java
Selector selector = Selector.open();
ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();

serverSocketChannel.configureBlocking(false);
serverSocketChannel.bind(new InetSocketAddress(port));

serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);

while (!Thread.currentThread().isInterrupted() && selector.select() > 0) { // accpet / read 할 수 있을 때까지 block
    final Iterator<SelectionKey> selectionKeyIterator = selector.selectedKeys().iterator();

    while (selectionKeyIterator.hasNext()) {
        final SelectionKey selectionKey = selectionKeyIterator.next();
        selectionKeyIterator.remove();

        if (!selectionKey.isValid()) {
            continue;
        }
        if (selectionKey.isAcceptable()) {
            ServerSocketChannel selectedChannel = (ServerSocketChannel)selectionKey.channel();

            final SocketChannel socketChannel = selectedChannel.accept(); // 연결이 들어온 상태이므로, 블로킹 없이 바로 소켓 채널을 취득한다. 
            if (socketChannel == null) {
                continue;
            }
            socketChannel.configureBlocking(false);
            socketChannel.register(selector, SelectionKey.OP_READ);
        } else if (selectionKey.isReadable()) {
            SocketChannel channel = (SocketChannel)selectionKey.channel();
            final ByteBuffer buffer = ByteBuffer.allocate(2048);

            final int bytesRead = channel.read(buffer); // 여기에서도 대기 없이, 바로 데이터를 읽는다.
            if (bytesRead == -1) {
                channel.close();
                selectionKey.cancel();
                continue;
            } else {
                buffer.flip();
                channel.write(buffer); // possible blocking. 코드 간략화를 위해 그냥 호출. 보통 os tcp buffer 에 바로 쓰여지므로 blocking 될 가능성은 낮다.
            }
        }
    }
}

selector.close();
```
[View in github](https://github.com/thywhite/tcp-echo-servers/blob/master/src/main/java/org/thywhite/springio/sharing/NonblockingServer.java)

### 설명
* Selector + NIO 를 조합한 EchoServer
* 코드의 양이 많이 늘었다.
* 소켓이 아닌 소켓 채널을 열고, 이 소켓 채널을 셀럭터에 등록한다.
  * 서버 채널은 연결 동작을 등록 - ```serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT)```
  * 클라이언트와 연결된 채널은 수신 동작을 등록 - ```socketChannel.register(selector, SelectionKey.OP_READ);```
* 셀렉터에서는 등록된 동작 등에 대해서, 블로킹 없이 바로 실행 가능한 동작을 알려주는 역할이다.
  * 여기에서 사용한 ```selector.select()``` 는 바로 실행 가능한 동작이 없으면, 있을때까지 대기(블로킹)한다.
  * 다른게 필요하면, ```selectNow(), select(long timeout)``` 을 사용할 수도 있다.
* select 된 이후에는, 모든 것이 Non blocking 으로 바로바로 처리 된다.
* 요약하면, 이벤트 드리븐 싱글 쓰레드 모델이다.
  * Node JS 가 자랑하던 바로 그것.

### select() 결과물을 처리하다가, 블로킹 메소드를 호출해버리면?
* Don't do it.
* 비동기 이벤트 기반 처리와 블로킹 메소드의 궁합은 심각하게 좋지 않다.
* 저 서버의 경우, ```selector.select()``` 가 호출이 늦어지게 된다.
* 이 말은, 다른 클라이언트가 연결을 했을 경우, 데이터를 보냈을 경우 이를 처리하는 것이 늦어지는 것을 의미한다.
* Netty나, 이를 기반으로 한 프레임워크 등에서, 블로킹 메소드를 호출하지 마세요라고 이야기하는 것은 이 때문.

### 블로킹 메소드 예시
* Thread.sleep()
* System.out.println()
* Stream.read() / write()
* synchronized block
* JDBC operation
* logger.log()
  * 설정에 따라서 async + discard 이면 OK. 
  * async 이더라도 discard 가 아닌 경우는 blocking 될 가능성 존재
* 기타 등등..

블로킹 메소드는 매우 많은데, 실수로 호출할 경우 파장이 크다.
그래서 Netty, Webflux 같은 프레임워크 사용 시에는 신경을 많이 써야 한다.

### 테스트 영상
![테스트 GIF](/img/tcp-echo-server/non-blocking-server.gif)
* [View in youtube](https://youtu.be/48O916I_-b0)

* 동시 접속 가능 테스트는 따로 수행하지 않음
* 다만 아주 잘 돌아감 

## Reactive Server

```java
public void start(int port) throws IOException {
    Publisher<SocketChannel> publisher = createSocketPublisher(port);

    publisher.subscribe(new Subscriber<SocketChannel>() {
        private final int nThreads = 2;
        private final ExecutorService publishOnExecutor = Executors.newFixedThreadPool(nThreads);
        private final ExecutorService subscribeOnExecutor = Executors.newSingleThreadExecutor();

        private final List<EchoProcessor> echoProcessors = new ArrayList<>();
        private Iterator<EchoProcessor> echoProcessorIterator;

        @SneakyThrows
        @Override
        public void onSubscribe(Subscription s) {
            for (int i = 0; i < nThreads; i++) {
                final EchoProcessor echoProcessor = new EchoProcessor();
                echoProcessors.add(echoProcessor);
                publishOnExecutor.submit(echoProcessor);
            }

            echoProcessorIterator = echoProcessors.iterator();
            subscribeOnExecutor.submit(() -> s.request(Long.MAX_VALUE)); // subscriber 가 publisher 에게 요청을 한다는 것이 차이점
        }

        @Override
        public void onNext(SocketChannel channel) {
            if (!echoProcessorIterator.hasNext()) {
                echoProcessorIterator = echoProcessors.iterator();
            }

            echoProcessorIterator.next().register(channel);
        }

        @Override
        public void onError(Throwable t) {

        }

        @Override
        public void onComplete() {

        }
    });
}

private Publisher<SocketChannel> createSocketPublisher(int port) throws IOException {
    Selector acceptSelector = Selector.open();
    ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();

    serverSocketChannel.configureBlocking(false);
    serverSocketChannel.bind(new InetSocketAddress(port));

    serverSocketChannel.register(acceptSelector, SelectionKey.OP_ACCEPT);

    return new Publisher<SocketChannel>() {
        @Override
        public void subscribe(Subscriber<? super SocketChannel> s) {
            s.onSubscribe(new Subscription() {
                @SneakyThrows
                @Override
                public void request(long n) {

                    while (acceptSelector.select() > 0) {
                        final Iterator<SelectionKey> keyIterator =
                                acceptSelector.selectedKeys().iterator();

                        while (keyIterator.hasNext()) {
                            final SelectionKey key = keyIterator.next();
                            keyIterator.remove();

                            final ServerSocketChannel serverSocketChannel =
                                    (ServerSocketChannel)key.channel();

                            final SocketChannel socketChannel = serverSocketChannel.accept();

                            socketChannel.configureBlocking(false);
                            s.onNext(socketChannel);
                        }
                    }
                }

                @SneakyThrows
                @Override
                public void cancel() {
                    acceptSelector.close();
                }
            });
        }
    };
}

private static class EchoProcessor implements Runnable {
    private final Selector selector;
    private final BlockingQueue<SocketChannel> channelBlockingQueue = new LinkedBlockingQueue<>();

    private EchoProcessor() throws IOException {
        selector = Selector.open();
    }

    @SuppressWarnings("Duplicates")
    @SneakyThrows
    @Override
    public void run() {
        for (; ; ) {
            SocketChannel newChannel;
            if ((newChannel = channelBlockingQueue.poll()) != null) {
                newChannel.register(selector, SelectionKey.OP_READ);
            }

            if (selector.select(100) > 0) {
                final Iterator<SelectionKey> keyIterator = selector.selectedKeys().iterator();

                while (keyIterator.hasNext()) {
                    final SelectionKey key = keyIterator.next();
                    keyIterator.remove();

                    SocketChannel channel = (SocketChannel)key.channel();
                    ByteBuffer buffer = ByteBuffer.allocate(2048);
                    final int bytesRead = channel.read(buffer);

                    if (bytesRead == -1) {
                        channel.close();
                        key.cancel();
                        continue;
                    } else {
                        buffer.flip();
                        channel.write(buffer);
                    }
                }
            }
        }
    }

    @SneakyThrows
    public void register(SocketChannel channel) {
        channelBlockingQueue.add(channel);
    }
}
```
[View in github](https://github.com/thywhite/tcp-echo-servers/blob/master/src/main/java/org/thywhite/springio/sharing/ReactiveServer.java)

### 설명
* 이제 리액티브 이야기.
* 이전 까지는 pure java만 사용했지만, 여기서부터는 [리액티브 스트림즈](https://github.com/reactive-streams/reactive-streams-jvm)를 사용하기 시작한다.

#### 리액티브 스트림즈 간단 요약
* publisher / subscriber 가 메인 컴포넌트
  * 예제에서는 SocketChannel 이 그 대상
* publisher 에 subscriber 를 등록하면, publisher 가 subscriber 한테 subscription 을 제공해준다.
* 이후 subscriber 에서 제공받은 subscription 을 사용해서, 데이터를 필요한만큼(처리 가능한만큼) 요청한다.
  * 예제에서는 죄다 처리하는 만큼 패기롭게 Long.MAX_VALUE 만큼 요청
* publisher 는 요청 받은 만큼 데이터를 subscriber 한테 보내려고 노력한다.
  * 예제에서는 ```accept()``` 할 때마다 하나씩 보낸다. request 받은 만큼만 보내는 부분은 지면 관계상 미구현이다.
* subscriber 는 받은 데이터를 필요한 데로 사용한다.
  * 예제에서는 ```EchoProcessor``` 에 등록한다.
  * ```EchoProcesoor``` 에서는 전달 받은 소켓을 등록하고, 자기 셀렉터의 등록한 후 Echo 로직을 실행한다.   
* 더 공급할 데이터가 없으면 complete 를 subscriber 에게 전달한다. 에러가 난 경우는 error 를 전달한다.
  * 예제에서는 아무것도 하지 않는다. 서버는 계속 실행되는것이 일반적이니만큼 
  
### 싱글 쓰레드가 아니네?
* 싱글 쓰레드에서 다시 멀티 쓰레드로 돌아왔다.
* subscribeOnExecutor -> 데이터를 요청 한 쓰레드. 이 쓰레드에서 Publisher 가 SocketChannel 을 공급해준다. 단일 쓰레드.
  * Netty 로 치면 Boss/Parent Group 이 하는 일이다.
* publishOnExecutor -> Publisher 가 데이터를 공급 한 쓰레드. 이 쓰레드에서 제공 받은된 SocketChannel 을 핸들링 한다. 멀티 쓰레드.
  * Netty 에서 그냥 Worker/Child Group 이 하는 일이다.  

* subscribeOn / publishOn 은 별도 Publisher(Operator 이기도 하다) 에서 처리하는 것이 적절하나, 코드 복잡도 상 Subscriber 에서 다 처리하게 해두었다.

* 사실 싱글 쓰레드 모델에서는, 멀티 코어 CPU 자원을 온전히 활용하기가 힘들다.
  * Node JS 에서는 클러스터를 통해서 이를 해결한다.
* 자바에서는 그냥, 핸들링 하는 쓰레드를 여러개 사용하면 된다.

* Netty 에서 bossGroup 기본값이 1, workerGroup 기본값이 CORE 수 * 2 인 이유도, 멀티코어 CPU를 풀로 활용하기 위해서다.

   
 ### 리액티브만의 장점은?
 
 * 사실 위 예제에서는 잘 느껴지지 않는다.
 * 주 차이점은 subscriber 에서 필요한만큼 데이터를 요청하는 부분이다.
 * 제대로 된 publiser 는 요청받은만큼만 데이터를 공급한다.
 * 빠른 publisher 가 느린 subscriber 에게 데이터를 계속 밀어넣다가 문제가 생기는 일이 원천 방지 된다.
 * 전문용어 : BACKPRESSURE
 
 * 이외에도 여러 장점이 있을지 모르겠지만, 내가 모름
 
 ### 테스트 영상
 
 * Non blocking 이랑 똑같으므로 생략
 
 
 ## 보너스
 * ReactiveServer 와 동일한 일을 하고 더 나은 로직
 
 ![Reactor-netty-version](/img/tcp-echo-server/reactor-netty.gif)
 
 
 끝.