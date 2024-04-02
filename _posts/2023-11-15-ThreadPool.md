---
title: ThreadPool
date: 2023-11-15 10:00 +0900
aliases: null
tags:
  - Java
  - ThreadPool
  - 쓰레드풀
  - ThreadPoolExecutor
  - corePoolSize
  - maximumPoolSize
  - ThreadPool Queue
  - RejectedExecutionPolicy
  - ScheduledExecutorService
image: /assets/img/thumbnails/Java.png
categories:
  - Skill
  - Java
---

## ThreadPool

어떤 요청이 들어왔을 때 해당 요청을 처리하기 위해서 쓰레드를 사용하는 가장 심플한 방법은 `요청마다 쓰레드를 생성하고 할당`하는 것이다

1. 쓰레드가 필요한 시점에 생성 요청
2. OS가 해당 쓰레드를 위한 메모리 영역 확보 및 할당
   - OS Level에서 Native Thread를 위한 메모리 영역을 할당
   - 생성된 Native Thread와 User Level Thread 매핑
     - Thread's start() 코드 내부에서 `JNI`를 통해서 Native Code 호출 (with C++)
3. 쓰레드 생성 및 Task 실행
4. ...
5. 쓰레드 사용이 끝나면 OS는 쓰레드를 위해 할당한 메모리 영역을 회수

<br>
이러한 과정이 매번 반복되고 결국 이러한 부분들이 쌓이게 되면 불필요한 리소스가 너무나도 많이 낭비가 되고 Application Performance에도 영향을 줄 수 있다

> 이러한 문제점을 해결하기 위해서 `ThreadPool`이 등장한다
{: .prompt-info }

<br>
<div style="text-align: left">
  <img src="/assets/img/posts/2023-11-15-ThreadPool/img1.png" alt="img"/>
</div>

ThreadPool은 필요할때마다 쓰레드를 생성/사용/반납하는게 아니라 <ins>미리 정해둔 개수만큼</ins> Pool에 쓰레드를 만들어놓고 `요청이 들어오면 만들어 놓은 쓰레드 중 Idle Thread를 할당`하는 방식이다

1. 미리 정해둔 개수만큼 ThreadPool에 Thread를 만들어놓는다
2. 요청이 들어오면 ThreadPool에서 쉬고 있는 Idle Thread 중 하나를 할당한다
   - 현재 Idle Thread가 없다면 요청을 Queue에 쌓아놓거나 Pool 상황 + 정책에 따라 거부할 수 있다
3. 할당된 Thread가 요청을 다 처리하면 다시 ThreadPool로 돌아간다

<br>
이렇게 Thread를 재사용함으로써 Thread 생성 비용 및 여러가지 불필요한 리소스 소모를 줄일 수 있다<br>
하지만 Thread 사용에 대한 Context를 지우지 않고 그대로 ThreadPool에 넣어버린다면 다른 요청에서 미처 지우지 못한 Context를 재사용함으로써 로직에 대한 오류가 발생할 수 있다<br>
- About `ThreadLocal`

### ThreadLocal을 사용할 때 주의점 (remove)

위에서 말했듯이 ThreadLocal을 통해서 `Thread별로 특정 자원을 관리`하는 경우 관련된 Thread들이 ThreadPool에서 `재사용`된다면 `엉뚱한 Context에서 이전 Thread가 작업하던 내용이 공유`될 수 있다

```kotlin
private val store = ThreadLocal<String>()

fun main() {
    val executor = Executors.newFixedThreadPool(1)
    val clients = listOf("ClientA", "ClientB")

    repeat(2) {
        executor.submit {
            println("## ${clients[it]} ##")
            println("첫번째 조회 = ${store.get()}")
            store.set("Private Value...")
            println("두번째 조회 = ${store.get()}\n")
        }
    }
}
```

ThreadLocal에 대한 사용이 종료되고 `remove`하지 않으면 어떤 문제가 발생하는지 코드를 통해서 체험해보자<br>
위의 코드는 `Executors.newFixedThreadPool(1)`를 통해서 ThreadPool에서 단 하나의 쓰레드만 유지하도록 적용하였다

> repeat간에 `동일한 쓰레드`를 재사용하기 위한 가정

- repeat1: ClientA 로직 진행
- repeat2: ClientB 로직 진행

```text
## ClientA ##
첫번째 조회 = null
두번째 조회 = Private Value...

## ClientB ##
첫번째 조회 = Private Value...
두번째 조회 = Private Value...
```

- ClientB는 분명히 처음 진입하였는데 `첫번째 조회`에서 ThreadLocal에 ClientA의 작업 내용물이 남아있다

이와 같이 분명히 서로 다른 요청 문맥임에도 불구하고 누군지도 모를 쓰레드가 사용한 내용물이 남아있는 문제가 발생할 수 있다

<br>

```kotlin
private val store = ThreadLocal<String>()

fun main() {
    val executor = Executors.newFixedThreadPool(1)
    val clients = listOf("ClientA", "ClientB")

    repeat(2) {
        executor.submit {
            println("## ${clients[it]} ##")
            println("첫번째 조회 = ${store.get()}")
            store.set("Private Value...")
            println("두번째 조회 = ${store.get()}\n")
            store.remove() // ThreadLocal remove
        }
    }
}
```

```text
## ClientA ##
첫번째 조회 = null
두번째 조회 = Private Value...

## ClientB ##
첫번째 조회 = null
두번째 조회 = Private Value...
```

- 이와 같이 ThreadLocal을 사용하고 나서 `반드시 remove`해줌으로써 쓰레드 풀을 사용하는 메커니즘에서 다른 쓰레드가 이전 작업물을 공유하는 일이 발생해서는 안된다

<br>

## ThreadPoolExecutor

<div style="text-align: left">
  <img src="/assets/img/posts/2023-11-15-ThreadPool/img2.png" alt="img"/>
</div>
<div style="text-align: left">
  <img src="/assets/img/posts/2023-11-15-ThreadPool/img3.png" alt="img"/>
</div>

> ThreadPoolExecutor는 요청으로 들어오는 여러 Task와 ThreadPool을 관리하기 위한 클래스이다
{: .prompt-tip }

### 주요 필드

#### 1. corePoolSize

<div style="text-align: left">
  <img src="/assets/img/posts/2023-11-15-ThreadPool/img4.png" alt="img"/>
</div>

> ThreadPool에서 `최소한으로 유지`되어야 하는 쓰레드 개수
{: .prompt-info }

- corePoolSize만큼의 Thread가 모두 일하고 있다면 추가적으로 들어오는 Task들은 workQueue에서 대기한다

#### 2. maximumPoolSize

<div style="text-align: left">
  <img src="/assets/img/posts/2023-11-15-ThreadPool/img5.png" alt="img"/>
</div>

> ThreadPool이 관리할 수 있는 `최대 쓰레드 개수`
{: .prompt-info }

- `workQueue가 가득 찬 경우` 큐에 쌓인 작업들은 `maximumPoolSize까지 동적으로 쓰레드를 생성해서 처리`할 수 있다
  - corePoolSize + **<span style="color:red">{x..}</span>** = maxmiumPoolSize

#### 3. keepAliveTime

<div style="text-align: left">
  <img src="/assets/img/posts/2023-11-15-ThreadPool/img6.png" alt="img"/>
</div>

> Idle 상태의 Thread가 `얼마동안 살아있을지`에 대한 시간
{: .prompt-info }

- corePoolSize로 부족해서 `동적으로 maximumPoolSize까지 생성될 수 있는 쓰레드가 생성`된 경우 해당 쓰레드가 Idle을 유지할 수 있는 시간
  - corePoolSize + **<span style="color:red">{x..}</span>** = maxmiumPoolSize
  - x.. 개수의 Thread에 대해서 적용되는 시간

#### 4. workQueue: `BlockingQueue<Runnable>`

<div style="text-align: left">
  <img src="/assets/img/posts/2023-11-15-ThreadPool/img7.png" alt="img"/>
</div>

> corePoolSize만큼의 Thread들이 모두 Task를 진행중일 경우 추가적인 Task들이 대기하는 Queue
{: .prompt-info }

- 작업을 처리할 수 있는 `Idle Thread가 없는 경우` 들어온 작업들은 workQueue에 들어가서 대기
- BlockingQueue에는 null Task 자체가 들어갈 수 없기 때문에 poll()을 통해서 가져온 Task도 null이 될 수 없다

### corePoolSize & maximumPoolSize & workQueue간의 관계

```kotlin
fun main() {
    val corePoolSize = 10
    val maximumPoolSize = 50
    val keepAliveTime = 10L
    val queue = LinkedBlockingQueue<Runnable>()
    val executor = ThreadPoolExecutor(
        corePoolSize,
        maximumPoolSize,
        keepAliveTime,
        TimeUnit.SECONDS,
        queue
    )

    val tasks = 60
    repeat(tasks) {
        executor.submit { Thread.sleep(1000L) }
    }
    repeat(tasks * 2) {
        Thread.sleep(500L)
        println("Active = ${executor.activeCount}")
        println("Queuing = ${queue.size}")
    }
}
```

이 코드는 과연 어떻게 동작할까?

##### [Case A]

1. corePoolSize = 10개의 Thread가 Task 1 ~ 10 진행
2. maximumPoolSize = 50이므로 Task 11 ~ 50에 대해서 Thread를 생성해서 진행
3. Task 51 ~ 60은 Queue에서 대기

##### [Case B]

1. corePoolSize = 10개의 Thread가 Task 1 ~ 10 진행
2. Task 11 ~ 60은 Queue에서 대기
3. 앞선 10개의 Task가 끝나면 Queue에서 대기하는 Task들에 대해서 10개씩 corePoolSize만큼의 Thread가 처리

<br>

> **<span style="color:red">Case A</span>**를 선택했다면 `틀린 방식`을 선택한 것이다

```text
Active = 10
Queuing = 50
Active = 10
Queuing = 40
Active = 10
Queuing = 40
Active = 10
Queuing = 30
Active = 10
Queuing = 30
Active = 10
Queuing = 20
Active = 10
Queuing = 20
Active = 10
Queuing = 10
Active = 10
Queuing = 10
Active = 10
Queuing = 0
Active = 10
Queuing = 0
Active = 0
...
```

- Case B와 같이 동작하는 것을 알 수 있다

<br>

<div style="text-align: left">
  <img src="/assets/img/posts/2023-11-15-ThreadPool/img8.png" alt="img"/>
</div>

1. corePoolSize보다 적은 Thread가 실행 중
   - 요청으로 들어온 Runnable Task를 처리할 `새로운 Thread를 생성`해서 실행
2. corePoolSize보다 많은 + maximumPoolSize보다 적은 Thread가 실행 중
   - Queue가 가득 찬 경우 = maximumPoolSize가 감당할 수준까지 새로운 Thread를 생성해서 실행
   - Queue가 여유로운 경우 = 해당 Task를 Queue에 넣어서 대기시키기

<div style="text-align: left">
  <img src="/assets/img/posts/2023-11-15-ThreadPool/img9.png" alt="img"/>
</div>

1. corePoolSize보다 적은 Thread가 실행 중
   - 무조건 새로운 Thread 생성해서 실행
2. corePoolSize보다 같거나 많은 Thread가 실행 중
   - 무조건 Queuing
3. 요청이 Queuing될 수 없는 경우
   - maximumPoolSize가 감당할 수준까지 새로운 Thread 생성해서 실행

<br>
ThreadPoolExecutor Docs에서 반복적으로 말하고 있는 우선순위 기준은 다음과 같다

> 1. corePoolSize <br>
> 2. Queuing <br>
> 3. maximumPoolSize
{: .prompt-tip }

그렇기 때문에 위의 케이스에서는 corePoolSize에 해당하는 Thread는 모두 실행중이고 `maximumPoolSize & Queue가 모두 여유가 있기 때문에 Queuing을 우선적으로 고려한 Case B`가 정답인 것이다

### maximumPoolSize & workQueue간의 관계

<div style="text-align: left">
  <img src="/assets/img/posts/2023-11-15-ThreadPool/img10.png" alt="img"/>
</div>

위의 예시코드에서 적용한 BlockingQueue이다<br>
코드를 보면 `Queue의 Capacity`가 ***<ins>Integer.MAX_VALUE</ins>***로 적용됨을 확인할 수 있다

<br>

> 다시 생각해보면 위의 코드에서 전체 Task가 60이고 Queue's Capacity는 Integer.MAX_VALUE인데 maximumPoolSize가 의미가 있을까?

결론은 maximumPoolSize는 사실상 의미가 없는 옵션이라고 생각할 수 있다

- Task가 60개밖에 없는데 Queue's Capacity는 훨씬 크기 때문에 maximumPoolSize 옵션은 소용이 없는 것

<br>
위의 예제 코드에서 `Queue's Capacity = 10`으로 설정하고 다시 시도해보자

```kotlin
val queue = LinkedBlockingQueue<Runnable>(10)
```

```text
Active = 50
Queuing = 10
Active = 10
Queuing = 0
Active = 10
Queuing = 0
Active = 0
Queuing = 0
...
```

이제서야 maximumPoolSize 옵션이 의미있게 적용됨을 확인할 수 있다

|                                                 |                                                   설명                                                   |                   Active & Queuing                   |
|:-----------------------------------------------:|:------------------------------------------------------------------------------------------------------:|:----------------------------------------------------:|
|                   Task 1 ~ 10                   |                            corePoolSize = 10이므로 Task 1 ~ 10은 해당 쓰레드에 바로 할당                             |              Active = 10<br>Queuing = 0              |
| **<span style="color:red">Task 11 ~ 20</span>** |                          corePoolSize만큼의 Thread가 모두 실행 중이기 때문에 Queue에 들어가서 대기                          |             Active = 10<br>Queuing = 10              |
|                  Task 21 ~ 30                   | corePoolSize만큼의 Thread가 모두 실행 중이고 Queue도 가득 찼기 찼음<br>-> maximumPoolSize에 의해서 동적으로 새로운 Thread가 생성되어서 실행 | Active = 20 (maximumPoolSize 20/50)<br>Queuing = 10  |
|                  Task 31 ~ 40                   | corePoolSize만큼의 Thread가 모두 실행 중이고 Queue도 가득 찼기 찼음<br>-> maximumPoolSize에 의해서 동적으로 새로운 Thread가 생성되어서 실행 | Active = 30  (maximumPoolSize 30/50)<br>Queuing = 10 |
|                  Task 41 ~ 50                   | corePoolSize만큼의 Thread가 모두 실행 중이고 Queue도 가득 찼기 찼음<br>-> maximumPoolSize에 의해서 동적으로 새로운 Thread가 생성되어서 실행 | Active = 40  (maximumPoolSize 40/50)<br>Queuing = 10 |
|                  Task 51 ~ 60                   | corePoolSize만큼의 Thread가 모두 실행 중이고 Queue도 가득 찼기 찼음<br>-> maximumPoolSize에 의해서 동적으로 새로운 Thread가 생성되어서 실행 | Active = 50  (maximumPoolSize 50/50)<br>Queuing = 10 |
| **<span style="color:red">Task 11 ~ 20</span>** |                             Thread가 여유로운 시점에 Queue에서 대기하고 있는 10개의 Task를 실행                             |             Active = 10<br>Queuing = 10              |

### RejectedExecutionHandler

또 다른 상황을 고려해보자

- Queue 가득 참
- maximumPoolSize만큼의 Thread 모두 실행 중

<br>
이 상황에서 추가적인 Task가 투입되면 어떤 일이 발생할까?

> 이 경우의 동작 방식은 RejectedExecutionHandler에 따라 달라진다
{: .prompt-info }

<div style="text-align: left">
  <img src="/assets/img/posts/2023-11-15-ThreadPool/img11.png" alt="img"/>
</div>

<br>

#### 1. AbortPolicy (Default)

<div style="text-align: left">
  <img src="/assets/img/posts/2023-11-15-ThreadPool/img12.png" alt="img"/>
</div>

- 더이상 Task를 처리할 수 없는데 추가적인 Task가 들어오는 경우 `RejectedExecutionException`를 발생시키는 정책

```kotlin
executor.rejectedExecutionHandler = ThreadPoolExecutor.AbortPolicy()
```

<div style="text-align: left">
  <img src="/assets/img/posts/2023-11-15-ThreadPool/img13.png" alt="img"/>
</div>

#### 2. CallerRunsPolicy

<div style="text-align: left">
  <img src="/assets/img/posts/2023-11-15-ThreadPool/img14.png" alt="img"/>
</div>

- Task를 execute한 Thread에서 처리되지 못한 Task까지 맡아서 실행하는 정책

```kotlin
executor.rejectedExecutionHandler = ThreadPoolExecutor.CallerRunsPolicy()
```

<div style="text-align: left">
  <img src="/assets/img/posts/2023-11-15-ThreadPool/img15.png" alt="img"/>
</div>

#### 3. DiscardPolicy

<div style="text-align: left">
  <img src="/assets/img/posts/2023-11-15-ThreadPool/img16.png" alt="img"/>
</div>

- 예외를 발생시키지도 않고 처리하지도 않고 그냥 무시하는 정책

```kotlin
executor.rejectedExecutionHandler = ThreadPoolExecutor.DiscardPolicy()
```


#### 4. DiscardOldestPolicy

<div style="text-align: left">
  <img src="/assets/img/posts/2023-11-15-ThreadPool/img17.png" alt="img"/>
</div>

- 처리되지 않은 가장 오래된 Task를 제거하고 현재 Task를 시도해보는 정책

```kotlin
executor.rejectedExecutionHandler = ThreadPoolExecutor.DiscardOldestPolicy()
```

<br>

## ScheduledExecutorService

> `지정된 시간 이후에 실행 or 주기적으로 실행`하도록 Task Command를 관리하는 ExecutorService
{: .prompt-tip }

- delay가 0이거나 음수일 경우 즉시 실행으로 간주

### schedule

<div style="text-align: left">
  <img src="/assets/img/posts/2023-11-15-ThreadPool/img18.png" alt="img"/>
</div>

- `주어진 Delay 후 단 1번의 실행만을 보장`하는 스케줄링 Task

```kotlin
val executor = ScheduledThreadPoolExecutor(5)

println("=== Start = ${LocalDateTime.now()} ===")
executor.schedule(
    {
        println("Thread = ${Thread.currentThread().name}")
        println("-> Time = ${LocalDateTime.now()}")
    },
    2,
    TimeUnit.SECONDS
)
```

<div style="text-align: left">
  <img src="/assets/img/posts/2023-11-15-ThreadPool/img19.png" alt="img"/>
</div>

- 주어진 Delay(2초) 후 정확히 1번만 실행

### scheduleAtFixedRate

<div style="text-align: left">
  <img src="/assets/img/posts/2023-11-15-ThreadPool/img20.png" alt="img"/>
</div>

- 처음 실행 = initialDelay가 지나고 실행
- 그 다음 실행 = Delay 간격으로 실행

```kotlin
val executor = ScheduledThreadPoolExecutor(5)

println("=== Start = ${LocalDateTime.now()} ===")
executor.scheduleAtFixedRate(
    {
        Thread.sleep(1000L)
        println("Thread = ${Thread.currentThread().name}")
        println("-> Time = ${LocalDateTime.now()}")
    },
    3,
    2,
    TimeUnit.SECONDS
)
```

<div style="text-align: left">
  <img src="/assets/img/posts/2023-11-15-ThreadPool/img21.png" alt="img"/>
</div>

- 처음 실행 = initialDelay(3초) + Thread.sleep(1000) = 4초 후
- 그 다음 실행 = Delay(2초) 간격

### scheduleWithFixedDelay

<div style="text-align: left">
  <img src="/assets/img/posts/2023-11-15-ThreadPool/img22.png" alt="img"/>
</div>

- 처음 실행 = initialDelay가 지나고 실행
- 그 다음 실행 = `이전 Command 종료 + Delay`가 지나고 실행

```kotlin
val executor = ScheduledThreadPoolExecutor(5)

println("=== Start = ${LocalDateTime.now()} ===")
executor.scheduleWithFixedDelay(
    {
        Thread.sleep(1000L)
        println("Thread = ${Thread.currentThread().name}")
        println("-> Time = ${LocalDateTime.now()}")
    },
    3,
    2,
    TimeUnit.SECONDS
)
```

<div style="text-align: left">
  <img src="/assets/img/posts/2023-11-15-ThreadPool/img23.png" alt="img"/>
</div>

- 처음 실행 = initialDelay(3초) + Thread.sleep(1000) = 4초 후
- 그 다음 실행 = 이전 실행 + Delay(2초) 간격으로 실행
  - 이전 실행 = Thread.sleep(1000) + sout
