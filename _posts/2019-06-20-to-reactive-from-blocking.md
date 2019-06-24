---
layout: post
title: 블로킹에서 리액티브 까지
permalink: /to-reactive-from-blocking
categories: [it]
tags: [reactive, java, tcp, network]
---

# 제목 : To Reactive From Blocking

블로킹에서 리액티브까지, 가즈아.

# 포스트 계기

이곳저곳에서 리액티브 이야기가 많이 들려온다.
사실, 여기에 관심을 갖게 된 것은 spring-webflux 때문이다.

spring-webmvc 로 빅히트를 친 전적이 있는, spring-framework 에서 새로 내놓은 웹 개발 프레임워크.
webmvc 놔두고, 왜 webflux 를 만들었고, 왜 또 그것을 그렇게 밀고 있을까?

과연 내 코딩 라이프에 webflux 가 webmvc 를 대체하게 될까?
지금 생각해보면, 이런 의구심으로 이 리액티브 세계에 발을 들이밀기 시작한 것 같다.

webflux 는 고도화가 많이 되어 있으므로, 이를 이해하기 위해 단순한데에서부터 시작해서,
왜 리액티브가 핫한지, 뭐가 좋은지, 쓸만은 한건지에 대해서 풀어볼려고 한다.

# 포스트 목표

* 리액티브가 뭐야? 라고 했을 때 대답할 수 있기.
* 그게 왜 좋아? 에 대답할 수 있기.

# Intro

[Node JS](https://nodejs.org/ko/about/) 소개 페이지에 아래와 같은 자랑이 있다.

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
    threadPool.submit(() -> { // 클라이언트 데이터 수신을 thread pool에 맡긴다. 
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
* ```serverSocket.accept()``` 만 하는 **메인** 쓰레드 하나, 클라이언트 별로 ```is.read(buffer)```를 수행 하는 쓰레드(쓰레드 풀에 들어있는) 들로 구성된다.
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
* **Don't do it.**
* 비동기 이벤트 기반 처리와 블로킹 메소드의 궁합은 심각하게 좋지 않다.
* 저 서버의 경우, ```selector.select()``` 가 호출이 늦어지게 된다.
* 이 말은, 다른 클라이언트가 연결을 했을 경우, 데이터를 보냈을 경우 이를 처리하는 것이 늦어지는 것을 의미한다.
* Netty나, 이를 기반으로 한 프레임워크 등에서, 블로킹 메소드를 호출하지 마세요라고 이야기하는 것은 이 때문.

### 블로킹 메소드 예시
* ```Thread.sleep()```
* ```System.out.println()```
* ```Stream.read() / write()```
* synchronized block
* JDBC operation
* ```logger.log()```
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
* 단순 에코 서버이고, 컨텍스트 스위칭 비용도 없으므로, 수만 커넥션은 문제 없을 것으로 생각.
  * 테스트는 해보지 않았다.
  * 이전 쓰레드 풀 기반 서버 보다 효율적인 것은 명백.


## Netty Server

```java
public void start(int port) throws InterruptedException {
    final NioEventLoopGroup bossGroup = new NioEventLoopGroup(1); // accept() 만 처리하는 보스 그룹
    final NioEventLoopGroup workerGroup = new NioEventLoopGroup(Runtime.getRuntime().availableProcessors()); // 에코 로직(read&write)만 처리하는 워커 그룹

    new ServerBootstrap()
            .group(bossGroup, workerGroup)
            .channel(NioServerSocketChannel.class)
            .childHandler(new ChannelInitializer<SocketChannel>() {
                @Override
                protected void initChannel(SocketChannel ch) throws Exception {
                    ch.pipeline().addLast(new EchoHandler());
                }
            })
            .bind(port)
            .channel().closeFuture().sync();
}

static class EchoHandler extends ChannelInboundHandlerAdapter {
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        ctx.channel().writeAndFlush(msg);
    }
}
```
[View in github](https://github.com/thywhite/tcp-echo-servers/blob/master/src/main/java/org/thywhite/springio/sharing/NettyServer.java)

### 설명
* 이제 리액티브 이야기를 시작하고 싶었지만, 저 긴 코드에다가 리액티브가 적용되면 코드가 너무 복잡해진다는 이야기를 들었으므로, 중간단계를 추가했다.
* 이전 까지는 pure java만 사용했지만, 여기서부터는 라이브러리를 사용하기 시작한다.
* Netty 를 사용해, 위 Non blocking server 를 압축했다. 위에 있는 pure java 코드 들은 ```NioServerSocketChannel```, ```NioSocketChannel``` 내부에 들어가게 된다.

### Netty Event Loop Group
* Netty 에서 사용하는 쓰레드 풀, 이제 더 이상 싱글 쓰레드가 아니다.
* 그렇다고 이전 쓰레드 풀처럼, 요청 마다 쓰레드를 할당하는 구조도 아니다.
* bossGroup 에서는 ```accept()``` 만 한다. accept 를 통해 할당된 채널(연결)들은 workerGroup 에 균등 분배된다.
  * 이거는 시간이 오래 걸리는 로직이 아니므로, 쓰레드 1개로도 차고 넘친다.
* workerGroup 에서는 ```read(), write()``` 를 한다.
  * 이전에는 10000 커넥션이 있더라도, 싱글쓰레드에서 모두 처리했다.
  * 코어수 4인 시스템을 가정하면, 이 NettyServer 에서는 쓰레드 당 2500 커넥션을 처리한다.
  * 멀티 코어 시스템에서는, 싱글 쓰레드 대비 더 빨리 처리할 수 있다는 이야기이다.

* 사실 싱글 쓰레드 모델에서는, 멀티 코어 CPU 자원을 온전히 활용하기가 힘들다.
  * Node JS 에서는 클러스터(멀티프로세스)를 통해서 이를 해결한다.
* 자바에서는 그냥, 핸들링 하는 쓰레드를 여러개 사용하면 된다. 지금 사용한 Netty 처럼.

* 사실 Netty 에서 bossGroup 기본값이 1, workerGroup 기본값이 CORE 수 * 2 인 이유도, 멀티코어 CPU를 풀로 활용하기 위해서다.
* 테스트 영상은 Non blocking server와 다르지 않으므로 생략.
  * 우린 이미 충분히 효율적인 에코 서버를 만들어냈다.

## Reactive Server

```java
public void start(int port) {
    Flux<Channel> channelFlux = Flux.<Channel>create(sink -> {
        NioEventLoopGroup bossGroup = new NioEventLoopGroup(1);
        NioEventLoopGroup workerGroup = new NioEventLoopGroup();

        new ServerBootstrap()
                .group(bossGroup, workerGroup)
                .channel(NioServerSocketChannel.class)
                .childHandler(new ChannelInitializer<SocketChannel>() {
                    @Override
                    protected void initChannel(SocketChannel ch) throws Exception {
                        // Flux 에 채널을 계속 공급
                        sink.next(ch);
                    }

                    @Override
                    public void channelInactive(ChannelHandlerContext ctx) throws Exception {
                        sink.complete();
                    }
                })
                .bind(port);
    }).log();

    Flux<Tuple2<Channel, ByteBuf>> channelAndReceivedFlux = channelFlux.flatMap(channel -> Flux.create(sink -> {
        channel.pipeline().addLast(new ChannelInboundHandlerAdapter() {
            @Override
            public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
                // Flux 에 채널이랑, 읽은 것을 계속 공급
                sink.next(Tuples.of(ctx.channel(), (ByteBuf)msg));
            }

            @Override
            public void channelInactive(ChannelHandlerContext ctx) throws Exception {
                // 연결이 끊어지면, 더 이상 보낼 것이 없으므로 완료 처리
                sink.complete();
            }
        });
    }));

    channelAndReceivedFlux.subscribe(new Subscriber<Tuple2<Channel, ByteBuf>>() {
        @Override
        public void onSubscribe(Subscription s) {
            // 계속 처리할 것이므로 최고값 요청
            s.request(Long.MAX_VALUE);
        }

        @Override
        public void onNext(Tuple2<Channel, ByteBuf> tuple) {
            Channel channel = tuple.getT1();
            ByteBuf received = tuple.getT2();
            channel.writeAndFlush(received);
        }

        @Override
        public void onError(Throwable t) {
        }

        @Override
        public void onComplete() {
        }
    });
}
```
[View in github](https://github.com/thywhite/tcp-echo-servers/blob/master/src/main/java/org/thywhite/springio/sharing/ReactiveServer.java)
   
### 설명
* 네티 서버에 비하면 많이 지저분해졌다.
* 깔금한 버전은 다음에 나올 것이다. 지금은 리액티브 개념을 설명하기 위해 위와 같은 구조로 변경하였다.

* 리액티브의 주 관심사는 비동기 데이터 스트림을 아주아주 잘 세련되게 처리하는 것이다.
* 그래서 위 네티 서버 로직을 비동기 데이터 스트림으로 표현한 것이 위 예제이다.

#### 첫번째 Flux - ```Flux<Channel> channelFlux```
* Flux는 비동기 데이터를 공급하는 역할을 한다.
  * Reactive 표준 인터페이스인 [reactive-streams](https://github.com/reactive-streams/reactive-streams-jvm) 에서는 Publisher 로 표현한다.
  * [project-reactor](https://github.com/reactor/reactor) 에서 이 Publisher 를 구현했고, 그 구현체 중 하나가 Flux 이다.
  * Flux 는 0~n 개의 데이터를 퍼블리싱 한다.
* 이 Flux 는 클라이언트와 연결이 이루어 질 때마다, Channel 을 계속 공급한다.
* ```List.stream()``` 이랑 비교하면 아래와 같은 차이점이 있다.
  1. 스트림 생성 시점 시, 어떤 데이터를 몇 개 줄지가 결정되어 있지 않다. (not predetermined)
  2. 스트림은 1회 용이다. 한번, count나 collect 등을 수행하면 재 사용할 수 없다. 반면 Flux 는 여러번 subscribe 를 할 수 있다.
  3. Stream 에서는 데이터를 계속 push 하는 방법만 있지만, Flux 에서는 데이터를 요청 할 수 있다. ```s.request(Long.MAX_VALUE)```
  4. 여기에서는 다루지 않지만 Flux 에서는 publish 하는 쓰레드, subscribe 를 하는 쓰레드 등을 쉽게 지정할 수 있다.

### 두번째 Flux - ```Flux<Tuple2<Channel, ByteBuf>> channelAndReceivedFlux```
* 첫번째 Flux 에 flatMap 연산을 적용한 결과이다.
* 여기에서는 채널이랑 채널에서 보낸 데이터를 조합한다.
* 채널에서 데이터를 받는것 또한 비동기 데이터 공급이므로, Flux 를 또 생성 한 것이다.
  * 첫번째는 보스 채널에서 비동기로 클라이언트 채널 생성
  * 두번째는 클라이언트 채널에서 비동기로 수신 데이터와 채널 생성

### Subscriber
* Flux 에서 공급받은 데이터를 가지고 뭔가를 하는 역할이다.
* 여기에서는 에코 로직을 수행한다.
* subscriber 는 publisher 로부터 subscription 을 제공 받고, 이 친구를 사용해서 데이터를 요청하고 요청한 데이터를 전달 받는다.
  * 데이터를 요청 -> ```s.request(Long.MAX_VALUE);```
  * 데이터를 제공 받음 -> ```onNext```
  * 데이터가 끝났다는 것을 전달 받음 -> ```onComplete```
  * 오류가 생겼다는 것을 전달 받음 -> ```onError```

* 여기에서 포인트는, 데이터를 요청한다는 것이다.
* publisher 가 제대로 된 친구라면, 요청한 만큼만 데이터를 공급한다.
* 빠른 publisher 가 느린 subscriber 에게 데이터를 계속 밀어넣다가 문제가 생기는 일이 원천 방지 된다.
* 전문용어 : BACKPRESSURE

지금은 단순 echo 로직이라 패기롭게 Long.MAX_VALUE 를 요청한 것이다. 반면, 위에 Netty 서버에 비슷한 로직을 추가하려면 별도의 인터셉터를 추가해야 한다.
그리고 그 추가한 로직을, 다른 비슷한 비동기 데이터 처리에 재활용할 수 있으리라는 보장도 없다.
반면, reactive-streams 위와 같이 backpressure 를 위한 표준 인터페이스가 규정되어 있으므로, 그대로 사용만 하면 된다.

애초에 reactive-streams 의 목적이 이것이다.
> The purpose of Reactive Streams is to provide a standard for asynchronous stream processing with non-blocking backpressure.

이제는 이 말이 이해가 갈 것이라고 믿는다.

자 이제 마지막으로..

## Reactor Netty Server
```java
public void start(int port) {
    TcpServer.create()
            .port(port)
            .handle((nettyInbound, nettyOutbound) -> {
                ByteBufFlux receive = nettyInbound.receive();
                NettyOutbound send = nettyOutbound.send((Publisher<ByteBuf>)receive.retain());
                return (Publisher<Void>)send;
            })
            // 1 줄로도 가능
            //.handle((nettyInbound, nettyOutbound) -> nettyOutbound.send(nettyInbound.receive().retain()))
            .bindNow();
}
```
[View in github](https://github.com/thywhite/tcp-echo-servers/blob/master/src/main/java/org/thywhite/springio/sharing/ReactiveNettyServer.java)

### 설명
* reactor-core 가 아닌 [reactor-netty](https://github.com/reactor/reactor-netty) 를 사용하면 위와 같은 코드가 가능하다.
* ReactiveServer 의 경우, 개념 설명을 위해 코드가 매우 방만한 것으로.. 실제로는 위와 같이 심플하게 구현이 가능하다.
* NettyServer 와 비교해도, 더 간결해보인다.
  * ServerBootstrap, WorkerGroup 설정 등이 안으로 다 들어가서 그렇다.
* Reactive Server 에서 Channler 과 ByteBuf를 받는 부분도, nettyInbound & nettyOutbound 로 잘 추상화되어있다.

* 중간에 있는 Publisher 로의 캐스팅은 사실 불필요하지만, 리액티브 개념설명을 위해 명시한 것이다.
* spring-webflux 에서도 Controller에서 Publisher 를 반환 하면 된다. 이후 spring-webflux 에서 해당 publisher에 subscriber 를 붙인다. 
* 이후 해당 Publisher에 데이터를 부어주면, spring-webflux 에서 그 데이터를 가지고 응답을 작성한다.
* 여기에 있는 TcpHandler에서 ```Publisher<Void>```를 반환하는 것도 유사하다. 다만 여기서는 데이터가 다른 채널로 이미 전달이 되었기 때문(```nettyOubbound.send```)에, 그 전달이 완료되었는지 여부만 핸들러에 전달하는 것이다.

## 이쯤에서 다시 보는 포스트 목표

* 리액티브가 뭐야? 라고 했을 때 대답할 수 있기
> 비동기 데이터 스트림을 잘 다루는 방법

* 그게 왜 좋아? 에 대답할 수 있기
> 시스템 자원을 아주 효율적으로 사용할 수 있으니까

# Next Post
* 이 Post 에서는 http가 아닌 tcp 를 다루었다.
* 그러므로 spring-webflux 이야기는 아주 조금 밖에 없는데, 이것이 아쉽다라는 요청이 있었으므로.. 아마 spring-webflux 이야기를 하지 않을까..?
* 언제 쓸지는 모르겠지만.. 이 블로그 포스트 간격을 보면 ㅋㅋㅋㅋ;

끝.