---
title: Tomcat 1 - Connector
date: 2023-11-16 10:00 +0900
aliases: null
tags:
  - Spring
  - Tomcat
  - Tomcat Connector
  - BIO NIO
  - Stream Channel
  - Selector
  - Worker Thread
  - Blocking Non-Blocking
image: /assets/img/thumbnails/Spring.png
categories:
  - Skill
  - Spring
---

이전 포스팅에서는 [ThreadPool](https://sjiwon.github.io/posts/ThreadPool/)에 대한 개념을 알아보았다<br>
본 포스팅에서는 `Tomcat의 Connector`에 대한 개념을 알아볼 것이다

- Tomcat:  WAS(Web Application Server)의 한 종류

<br>

## Connector란?

[Apache Tomcat Connector Docs](https://tomcat.apache.org/tomcat-10.0-doc/config/http.html#Introduction)

> The HTTP Connector element represents a Connector component that <span style="color:red">supports the HTTP/1.1 protocol.</span><br>
> It enables Catalina to function as a stand-alone web server, in addition to its ability to execute servlets and JSP pages.<br>
> A particular instance of this component <span style="color:red">listens for connections on a specific TCP port number on the server.</span><br>
> One or more such Connectors can be configured as part of a single Service, each forwarding to the associated Engine to perform request processing and create the response.
{: .prompt-info }

HTTP Connector는 Tomcat으로 요청이 들어올 때 거쳐가는 첫번째 관문으로써 가장 핵심이 되는 역할은 다음과 같다

1. 특정 TCP Port로부터 들어오는 Socket Connection들을 Listen
2. Socket Connection으로부터 데이터 패킷 파싱
3. 파싱한 데이터 패킷을 ServletRequest로 Wrapping해서 Servlet Container로 넘겨주기

<div style="text-align: left">
  <img src="/assets/img/posts/2023-11-16-Tomcat%201%20-%20Connector/img1.png" alt="img"/>
</div>

<br>

> Each incoming, non-asynchronous request requires a thread for the duration of that request.<br>
> If `more simultaneous requests` are received than can be handled by the currently available request processing threads, additional threads <span style="color:red">will be created up to the configured maximum</span> (the value of the ***maxThreads*** attribute).<br>
> <br>
> If `still more simultaneous requests are received`, Tomcat <span style="color:red">will accept new connections until the current number of connections reaches maxConnections.</span><br>
> Connections <span style="color:red">are queued</span> inside the server socket <span style="color:red">created by the Connector</span> until <span style="color:red">a thread becomes available to process the connection.</span><br>
> <br>
> Once `maxConnections has been reached` the operating system will queue further connections.<br>
> The size of the <span style="color:red">operating system provided connection queue</span> may be controlled by the ***acceptCount*** attribute.<br>
> If the operating system queue fills, further connection requests may be <span style="color:red">refused or may time out.</span>

특정 TCP Port를 Listening하다가 Connection이 들어오기 시작하면 해당 Connection을 처리해줄 Thread가 필요할 것이다

1. ***<ins>minSpare</ins>***만큼의 Thread로는 더이상 처리할 수 없는 많은 요청이 추가적으로 들어오면
   - <span style="color:green">maxThreads</span> 설정을 참고해서 추가적인 Thread를 생성해서 처리
2. ***<ins>maxThreads</ins>***만큼 늘려도 더 많은 요청들이 추가적으로 들어오면
   - <span style="color:green">maxConnections</span> 설정을 참고해서 일단 Accept해놓는다
   - Accept해놓은 Connection들은 `Connector Queue`에서 관리되다가 여유가 생긴 Thread한테 할당되어서 처리된다
3. ***<ins>maxConnection</ins>***까지 도달했다면
   - <span style="color:green">acceptCount</span> 설정을 참고해서 `OS Level Queue`에서 관리된다
   - 만약 OS Level Queue도 가득차서 더이상 받아들일 수 없다면 `Connection refused or timeout`

<br>

여기까지 설정값에 따른 전체적인 프로세스를 요약해보면 다음과 같다

> 1. min-spare만큼의 Thread(Active든 Idle이든)는 항상 유지된다<br>
> 2. min-spare만큼의 Thread가 모두 Active하다면<br>
>    - maxThreads까지 동적으로 Thread를 생성해서 Task에 할당
> 3. maxThreads까지 모두 Active하다면<br>
>    - maxConnections까지 Connection들을 Connector Queue에 Accept
>    - Queuing Connection들은 여유가 생긴 Thread에 의해서 처리
> 4. maxConnection에 대한 Connector Queue가 가득찼다면<br>
>    - acceptCount만큼의 Connection들을 OS Level Queue에 Accept
> 5. acceptCount에 대한 OS Level Queue도 가득찼다면
>    - Refused or Connection Timeout
{: .prompt-tip }

<br>

## BIO Connector

BIO Connector는 Socket Connection을 처리하기 위해서 `Blocking I/O`를 활용한다

<div style="text-align: left">
  <img src="/assets/img/posts/2023-11-16-Tomcat%201%20-%20Connector/img2.png" alt="img"/>
</div>

1. ThreadPool에서 대기중인 Thread는 Socket Connection에 할당된다
2. 이후 요청에 대한 처리 진행...
3. Socket Connection이 종료되면서 Thread는 다시 ThreadPool로 되돌아간다

> 이 과정에서 가장 중요하게 봐야할 부분은 `Socket Connection의 전체 생명주기동안` 특정 Thread를 계속 물리고 있다는 점이다<br>
> :: One Worker Thread per Socket Connection

따라서 이렇게 물려버린 특정 Thread는 해당 Socket Connection의 생명주기동안 다른 작업에 관여할 수 없게 된다<br>
이는 리소스의 낭비라고 볼 수 있다

<br>
예를 들어서 식당에 3명의 손님과 3명의 알바가 있다고 하자<br>
이 시점에는 손님 1 - 알바 1로 배정하면 되니까 아무 문제도 없을것이다<br>
그런데 여기서 손님 5명이 추가로 들어왔다고 하고 기존 손님 3명은 아직 식사중이다<br>
`BIO Connector`에 따르면 기존 손님 3명이 들어오고 식사하고 나갈때까지 3명의 알바는 계속 기존 손님들에게 배정되어 있을 것이다<br>

- 그러면 추가로 들어온 손님 5명은?
- 담당할 알바가 없다

알바 3명이 기존 손님들에게 배정되어 있는동안 어떤 일도 안하고 그냥 테이블 옆에서 대기만 하고 있다<br>
이러한 프로세스가 과연 알바를 효율적으로 사용하고 있다고 볼 수 있을까?

- 옆에서 대기만 하지말고 그 시간에 다른 손님들에게 배정되어서 왔다갔다 일을 하면 더 효율적으로 사용하는게 아닐까?

<br>

BIO Connector는 이러한 메커니즘으로 인해 리소스를 낭비한다<br>
따라서 이러한 문제점을 해결하기 위해서 `NIO Connector`가 등장한 것이다

### BIO Connector 아키텍처

NIO Connector로 넘어가기 전에 BIO Connector의 간단한 아키텍처를 살펴보자

<div style="text-align: left">
  <img src="/assets/img/posts/2023-11-16-Tomcat%201%20-%20Connector/img3.png" alt="img"/>
</div>

1. Acceptor는 Socket Connection을 획득하고 Worker Thread Pool에서 Idle 상태로 대기하고 있는 Thread를 찾아본다
   - Idle Thread가 없는 경우 현재 들어온 Connection을 처리할 수 없기 때문에 `Acceptor는 Block`된다 (동기적 처리)
2. Connection을 처리할 Worker Thread가 존재하면 Http11Processor를 통해서 요청을 처리한다
3. Http11Processor는 요청을 분석하고 Servlet에서 처리할 수 있도록 Wrapping한다
4. Wrapping 처리된 Connection은 CoyoteAdapter를 통해서 ServletContainer로 전달된다
5. ...
6. 요청에 대한 처리가 완료되면 응답되고 Socket Connection이 해제되면 이 순간에 Worker Thread는 Thread Pool로 돌아가서 Idle 상태가 된다

위에서도 언급했듯이 BIO Connector에서 가장 중요한 부분이자 문제점인 것은 `Socket Connection - Worker Thread가 1:1로 대응`된다는 점이다<br>
이러한 특징으로 인해 위의 예시와 같은 리소스 낭비가 발생하고 동시에 처리할 수 있는 Connection 역시 한계가 존재한다

<br>
[JIoEndpoint Docs](https://tomcat.apache.org/tomcat-8.0-doc/api/org/apache/tomcat/util/net/JIoEndpoint.html)

- Handle incoming TCP connections. 
- This class implement a simple server model: 
  - <span style="color:red">one listener thread accepts on a socket and creates a new worker thread for each incoming connection.</span>
  - More advanced Endpoints will reuse the threads, use queues, etc.

<br>

## NIO Connector

NIO Connector는 BIO Connector의 비효율적인 리소스 사용을 해결하고 `Java NIO (NIO = Non Blocking + Blocking)` 기반으로 Socket Connection들을 처리한다

### Buffer & Channel & Selector

#### 1. Buffer

NIO Buffer는 `데이터를 담아두는 컨테이너` 개념이고 NIO 메커니즘에서 모든 데이터들은 Buffer를 거쳐서 흘러다닌다

- Stream
  - 동기 + Blocking 방식으로 데이터를 `단방향`으로 처리
  - Byte 단위로 데이터를 즉시 전송
    - 만약 대량의 데이터를 보내게 된다면 매번 Byte 단위로 보내기 때문에 Network I/O에 대한 오버헤드가 심해질 수 있다
- Buffer
  - Stream과는 다르게 Byte 단위의 데이터를 매번 즉시 보내는게 아니라 `어느정도 모아서 한번에 보냄`으로써 I/O에 대한 오버헤드 감소 + 성능 향상

```kotlin
private const val BASE_PATH = "src/main/kotlin/com/sjiwon/kotlinplayground/io/"

fun copyWithStream(
    source: String,
    dest: String,
) {
    FileInputStream(BASE_PATH + source).use { input ->
        FileOutputStream(BASE_PATH + dest).use { output ->
            while (true) {
                val read = input.read()
                if (read == -1) {
                    break
                }
                output.write(read)
            }
        }
    }
}

fun copyWithBuffer(
    source: String,
    dest: String,
) {
    val inputChannel = FileInputStream(BASE_PATH + source).channel
    val outputChannel = FileOutputStream(BASE_PATH + dest).channel
    val buffer = ByteBuffer.allocate(1024)

    while (inputChannel.read(buffer) != -1) {
        buffer.flip()
        outputChannel.write(buffer)
        buffer.clear()
    }

    inputChannel.close()
    outputChannel.close()
}

fun execute(
    with: String,
    logic: () -> Unit,
) {
    val start = System.currentTimeMillis()
    logic()
    val end = System.currentTimeMillis()
    println("[With $with] -> Execute = ${end - start}ms")
}

fun main() {
    execute("Stream") { copyWithStream(source = "source.png", dest = "dest_stream.png") }
    execute("Buffer") { copyWithBuffer(source = "source.png", dest = "dest_buffer.png") }
}
```

<div style="text-align: left">
  <img src="/assets/img/posts/2023-11-16-Tomcat%201%20-%20Connector/img4.png" alt="img"/>
</div>

- 224KB 크기의 이미지를 각각 Stream & Buffer로 복사한 결과 Buffer의 성능이 압도적임을 확인할 수 있다

#### 2. Channel

NIO Channel은 `데이터가 흘러다니는 양방향 통로`이고 반드시 NIO Buffer를 통해서만 Read/Write가 가능하다<br>
또한 Non-Blocking I/O 기반으로 데이터를 처리할 수 있어서 쓰레드를 효과적으로 사용할 수 있다

<div style="text-align: left">
  <img src="/assets/img/posts/2023-11-16-Tomcat%201%20-%20Connector/img5.png" alt="img"/>
</div>

#### 3. Selector

`하나의 쓰레드`에서 `N개의 채널을 모니터링`하고 준비된 채널의 데이터를 처리할 수 있게 해주는 `Multiplexing 컴포넌트`이고 NIO Non-Blocking I/O의 핵심 요소이다

<div style="text-align: left">
  <img src="/assets/img/posts/2023-11-16-Tomcat%201%20-%20Connector/img6.png" alt="img"/>
</div>

1. 하나 이상의 Channel을 Selector에 등록한다
2. Thread가 `select()`를 호출하면 등록된 N개의 Channel 중 이벤트 준비가 완료된 Channel이 생길때까지 Block
3. 이후 select로부터 결과가 반환되면 이벤트 준비가 완료된 Channel이 있다는 의미이고 해당 Channel로부터 준비된 이벤트를 처리
  - select = 이벤트 준비가 완료된 Channel이 하나 이상 생길때까지 Block
  - selectNow = 준비된 Channel이 있으면 즉시 반환하고 select와는 달리 없어도 Block하지 않는다

### NIO Connector 아키텍처

#### 1. Acceptor

> SocketConnection을 Accept하는 Thread

1. <ins>NioEndpoint#serverSocketAccept</ins>를 통해서 SocketConnection을 Accept하고 SocketChannel을 획득한다
2. 획득한 SocketChannel을 Wrapping해서 <span style="color:red">PollerEventQueue에 Produce</span>한다

#### 2. Poller

> `Selector를 관리`하고 PollerEventQueue를 가지고 있는 Thread

<div style="text-align: left">
  <img src="/assets/img/posts/2023-11-16-Tomcat%201%20-%20Connector/img7.png" alt="img"/>
</div>

1. Loop를 돌면서 등록된 Event가 존재하는지 확인한다
2. <span style="color:red">select & selectNow</span>를 통해서 이벤트 준비가 완료된 Channel이 존재하는지 확인한다
3. 준비가 완료된 Channel이 하나 이상 존재한다면 해당 채널로부터 `SocketEvent`를 받아서 Worker Thread에게 넘긴다

#### 3. Worker

> Poller로부터 SocketEvent를 받아서 `SocketProcessor`로 Wrapping한 후 처리 시작

<div style="text-align: left">
  <img src="/assets/img/posts/2023-11-16-Tomcat%201%20-%20Connector/img8.png" alt="img"/>
</div>

### NIO Flow 정리

<div style="text-align: left">
  <img src="/assets/img/posts/2023-11-16-Tomcat%201%20-%20Connector/img9.png" alt="img"/>
</div>

NIO Connector를 정리해보면 `NIO Selector`라는 핵심 컴포넌트를 통해서 <span style="color:red">Multiplexing 기반</span>으로 N개의 Channel을 모니터링하다가 준비 완료된 Channel이 생기면 그 시점에 Event를 Worker Thread에게 넘겨서 처리하는 구조이다

<br>
여기서 BIO Connector와 비교해보자

- BIO Connector
  - Socket Connection이 들어온 시점부터 Worker Thread를 할당해서 모든 생명주기를 함께한다
  - Idle Worker Thread가 없는 경우 Socket Connection을 처리할 수 없기 때문에 Acceptor는 Blocking
  - 요청에 대한 응답이 끝나더라도 Socket Connection이 완전히 close될때까지 할당되어 있기 때문에 이 자체가 비효율적
- NIO Connector
  - <span style="color:red">데이터 처리가 가능한 시점</span>에 Worker Thread를 할당해서 처리
  - 처리가 끝나면 그 후 Socket Connection이 close될때까지 기다리지 않아도 되기 때문에 Idle로 낭비되는 시간을 줄일 수 있다

<div style="text-align: left">
  <img src="/assets/img/posts/2023-11-16-Tomcat%201%20-%20Connector/img10.png" alt="img"/>
</div>
