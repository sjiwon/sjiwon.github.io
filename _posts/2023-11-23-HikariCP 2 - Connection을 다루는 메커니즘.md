---
title: HikariCP 2 - Connection을 다루는 메커니즘
date: 2023-11-23 13:00 +0900
aliases: null
tags:
  - Spring
  - HikariCP
  - ConcurrentBag borrow()
  - sharedList
  - threadList
  - handoffQueue
  - HikariCP 옵션
image: /assets/img/thumbnails/Spring.png
categories:
  - Skill
  - Spring
---

[이전 포스팅](https://sjiwon.github.io/posts/HikariCP-1-DBCP/)에서 기본적인 DBCP에 대한 개념과 HikariCP의 핵심 컴포넌트에 대한 기본적인 개념을 알아보았다<br>
본 포스팅에서는 `HikariCP에서 실질적으로 DB Connection을 다루는 메커니즘`에 대해서 알아볼 것이다

<br>

## HikariCP 주요 필드

<div style="text-align: left">
  <img src="/assets/img/posts/2023-11-23-HikariCP%202%20-%20Connection을%20다루는%20메커니즘/img1.png" alt="img"/>
</div>

|                                      필드                                       | 설명                                                                                          |
|:-----------------------------------------------------------------------------:|:--------------------------------------------------------------------------------------------|
|                                   poolState                                   | HikariCP의 상태 [POOL_NORMAL / POOL_SUSPENDED / POOL_SHUTDOWN]                                 |
|                               poolEntryCreator                                | poolEntry에 새로운 Connection을 생성하기 위한 컴포넌트                                                     |
|                           postFillPoolEntryCreator                            | PoolEntry의 Idle Connection이 모자랄 경우 채우기 위한 컴포넌트                                              |
|               addConnectionExecutor<br>closeConnectionExecutor                | Connection을 add/close 하기 위한 Task를 관리                                                        |
|                 <span style="color:red">connectionBag</span>                  | Connection을 관리하는 가방 개념                                                                      |
|                               suspendResumeLock                               | HikariCP에서 Connection을 다루기 위해서 일시적으로 중단/재개하기 위한 Lock                                        |
| <span style="color:red">houseKeepingExecutorService</span><br>houseKeeperTask | HikariCP의 Idle Connection에 대한 timeout, maximum, ..등을 관리하고 <ins>minimum connection을 유지</ins> |


<br>

## HikariCP 메커니즘

### 1. HikariPool 초기화

#### 1) HouseKeeper 관련 ExecutorService 초기화, 테스트 연결 시도

<div style="text-align: left">
  <img src="/assets/img/posts/2023-11-23-HikariCP%202%20-%20Connection을%20다루는%20메커니즘/img2.png" alt="img"/>
</div>

먼저 HIkariCP 생성자를 통해서 초기화를 진행하게 되는데 핵심은 아래 로직이다

- initializeHouseKeepingExecutorService()
- <span style="color:red">checkFailFast()</span>

##### **_✏️ initializeHouseKeepingExecutorService()_**

<div style="text-align: left">
  <img src="/assets/img/posts/2023-11-23-HikariCP%202%20-%20Connection을%20다루는%20메커니즘/img3.png" alt="img"/>
</div>

DBCP의 Idle/Minimum Connection들을 관리하는 HouseKeeper를 주기적으로 동작시킬 ScheduledExecutorService를 초기화한다

- corePoolSize = 1개인 ThreadPool
- RejectedExecutionPolicy = DiscardPolicy
  - Queue, maximumPoolSize가 over되면 다음 Task는 버리기

##### **_✏️ checkFailFast()_**

<div style="text-align: left">
  <img src="/assets/img/posts/2023-11-23-HikariCP%202%20-%20Connection을%20다루는%20메커니즘/img4.png" alt="img"/>
</div>

그 이후 checkFailFast를 통해서 설정한 DBMS와 연결이 가능한지 먼저 확인을 진행한다

- [이전 포스팅](https://sjiwon.github.io/posts/HikariCP-1-DBCP/)에서 알아봤듯이 PoolEntry는 Connection을 Wrapping하고 추가적인 메타 정보를 트래킹하는 컴포넌트이다
- checkFailFast에서는 실제 DB Connection을 하나 얻게된다

<div style="text-align: left">
  <img src="/assets/img/posts/2023-11-23-HikariCP%202%20-%20Connection을%20다루는%20메커니즘/img5.png" alt="img"/>
</div>
<div style="text-align: left">
  <img src="/assets/img/posts/2023-11-23-HikariCP%202%20-%20Connection을%20다루는%20메커니즘/img6.png" alt="img"/>
</div>
<div style="text-align: left">
  <img src="/assets/img/posts/2023-11-23-HikariCP%202%20-%20Connection을%20다루는%20메커니즘/img7.png" alt="img"/>
</div>
<div style="text-align: left">
  <img src="/assets/img/posts/2023-11-23-HikariCP%202%20-%20Connection을%20다루는%20메커니즘/img8.png" alt="img"/>
</div>

위의 일련의 과정을 통해서 실제로 DB Connection을 가져오는데 이 결과에 따라 다음과 같은 로직으로 분기된다

<span style="color:green">※ PoolEntry 확보 O = Connection 획득 O</span>

① minimumIdle 값이 0보다 큰지

- DB 연결 테스트를 위해서 가져온 Connection을 `ConnectionBag`에 넣어준다
  - 테스트라고 하더라도 실제 DB Connection을 얻은거고 minimumIdle 또한 0보다 크기 때문에 유지할 가치가 있다고 판단

② minimumIdle 값이 0보다 작다면

- 유지해야 할 최소 Connection이 없다는 의미이므로 얻은 DB Connection을 닫아준다

<span style="color:green">※ PoolEntry 확보 X = Connection 획득 X</span>

Connection을 얻는 과정에서 ConnectionSetupException이 발생한지 확인한다

- 발생 O = 즉시 테스트 종료 및 예외 발생
- 발생 X = initializationTimeout이 다 소모될때까지 지속적으로 DB Connection을 얻으려는 시도 진행

#### 2) 여러 메트릭 초기화 및 Executor & HouseKeeper 설정

<div style="text-align: left">
  <img src="/assets/img/posts/2023-11-23-HikariCP%202%20-%20Connection을%20다루는%20메커니즘/img9.png" alt="img"/>
</div>

checkFailFast()를 통해서 테스트 DB 연결을 성공했다면 그 이후는 여러 메트릭 설정과 Executor & HouseKeeper 설저응ㄹ 진행한다

- addConnectionQueue는 Connection을 새로 만들기 위해서 관련된 Thread들을 대기시키는 Queue

#### 3) HouseKeeper minimumIdleConnection 채우기

<div style="text-align: left">
  <img src="/assets/img/posts/2023-11-23-HikariCP%202%20-%20Connection을%20다루는%20메커니즘/img10.png" alt="img"/>
</div>

HikariPool 초기화 마지막 단계로써 <span style="color:red">minimumIdle까지 Connection을 채우는 작업</span>을 진행하게 된다<br>
그런데 의문이 들 것이다

- 그 어디에도 Connection을 채우는 코드가 보이지 않음
- 그대신 `getTotalConnections() < config.getMinimumIdle()`을 통해서 minimumIdle이 다 채워졌는지 확인하는 코드는 보임

> 이 과정은 이전에 초기화 및 설정을 진행한 <span style="color:red">HouseKeeper</span>가 담당하게 된다
{: .prompt-info }

<br>

<div style="text-align: left">
  <img src="/assets/img/posts/2023-11-23-HikariCP%202%20-%20Connection을%20다루는%20메커니즘/img11.png" alt="img"/>
</div>
<div style="text-align: left">
  <img src="/assets/img/posts/2023-11-23-HikariCP%202%20-%20Connection을%20다루는%20메커니즘/img12.png" alt="img"/>
</div>

- [ScheduledExecutorService's scheduleWithFixedDelay](https://sjiwon.github.io/posts/ThreadPool/#scheduledexecutorservice)를 통해서 HouseKeeper의 작업을 관리하고 있다
  - 초기 Delay = 100ms
  - 작업 간 Delay = housekeepingPeriodMs = 이전 작업 진행시간 + 30ms

<div style="text-align: left">
  <img src="/assets/img/posts/2023-11-23-HikariCP%202%20-%20Connection을%20다루는%20메커니즘/img13.png" alt="img"/>
</div>
<div style="text-align: left">
  <img src="/assets/img/posts/2023-11-23-HikariCP%202%20-%20Connection을%20다루는%20메커니즘/img14.png" alt="img"/>
</div>

1. minimumIdle을 초과했다면 STATE_NOT_IN_USE나 유효하지 않은 Connection들 정리
2. minimumIdle이 모자라면 fillPool(true)를 통해서 채워주기

### 2. Connection 획득

#### 1) HikariDataSource#getConnection()

<div style="text-align: left">
  <img src="/assets/img/posts/2023-11-23-HikariCP%202%20-%20Connection을%20다루는%20메커니즘/img15.png" alt="img"/>
</div>
<div style="text-align: left">
  <img src="/assets/img/posts/2023-11-23-HikariCP%202%20-%20Connection을%20다루는%20메커니즘/img16.png" alt="img"/>
</div>

- HikariDataSource는 HikariConfig의 여러 설정값에 대해서 이 값들이 제대로된 값인지 검증한다 (URL, Driver, ...)
- 그 이후 pool & fastPathPool을 초기화한다

<div style="text-align: left">
  <img src="/assets/img/posts/2023-11-23-HikariCP%202%20-%20Connection을%20다루는%20메커니즘/img17.png" alt="img"/>
</div>

1. HikariDataSource차 close 상태인지 확인 
   - close면 Connection 획득 불가능
2. fastPathPool이 존재하면 fastPathPool을 통해서 getConnection()
3. fastPathPool이 없다면 그냥 pool을 통해서 getConnection()
4. fastPathPool & pool 둘다 없다면 HikariPool 새로 생성해서 getConnection()

#### 2) HikariPool#getConnection()

<div style="text-align: left">
  <img src="/assets/img/posts/2023-11-23-HikariCP%202%20-%20Connection을%20다루는%20메커니즘/img18.png" alt="img"/>
</div>

- HikariPool은 `connectionTimeout 이내에` DB Connection을 얻으려고 시도한다

##### 2-1) ConcurrentBag#borrow()

<div style="text-align: left">
  <img src="/assets/img/posts/2023-11-23-HikariCP%202%20-%20Connection을%20다루는%20메커니즘/img19.png" alt="img"/>
</div>

- connectionBag(ConcurrentBag)의 borrow를 통해서 DBCP에 존재하는 Connection을 가져오려는 시도를 진행한다

<br>

###### **_<span style="color:green">① threadList 확인</span>_**

<div style="text-align: left">
  <img src="/assets/img/posts/2023-11-23-HikariCP%202%20-%20Connection을%20다루는%20메커니즘/img20.png" alt="img"/>
</div>
<div style="text-align: left">
  <img src="/assets/img/posts/2023-11-23-HikariCP%202%20-%20Connection을%20다루는%20메커니즘/img21.png" alt="img"/>
</div>

> ThreadLocal에서 <span style="color:red">해당 Thread가 이전에 사용했던 Connection들 중에서 사용 가능한 Connection이 있다면</span> 해당 Connection을 사용
{: .prompt-info }

- compareAndSet을 통해서 STATE_NOT_IN_USE → STATE_IN_USE로 사용하겠다는 상태로 변경

<br>

###### **_<span style="color:green">② sharedList 확인</span>_**

<div style="text-align: left">
  <img src="/assets/img/posts/2023-11-23-HikariCP%202%20-%20Connection을%20다루는%20메커니즘/img22.png" alt="img"/>
</div>

이전에 자신이 사용했던 Connection을 캐싱하는 threadList에 사용 가능한 Connection이 없다면

- 이제는 `모든 Thread가 공유하는 HikariCP의 전체 Connection을 관리하는 sharedList`에서 찾아본다

- 만약 sharedList에 사용 가능한 Connection이 존재한다고 하더라도 `나보다 이전에 Connection을 기다리고 있던 waiter가 존재`한다면 waiter들을 위해서 listener에 Connection 생성 요청을 넣어두고 대기한다

<div style="text-align: left">
  <img src="/assets/img/posts/2023-11-23-HikariCP%202%20-%20Connection을%20다루는%20메커니즘/img23.png" alt="img"/>
</div>

> 전체 Connection을 보관하는 <span style="color:red">sharedList에서 사용 가능한 Connection을 찾아보고</span> 만약 있다면 대기자들도 체크하고 가져온다
{: .prompt-info }

<br>

###### **_<span style="color:green">③ handoffQueue에서 대기 (with Polling)</span>_**

<div style="text-align: left">
  <img src="/assets/img/posts/2023-11-23-HikariCP%202%20-%20Connection을%20다루는%20메커니즘/img24.png" alt="img"/>
</div>

threadList, sharedList 둘다 사용 가능한 Connection이 없다면

- handoffQueue를 지속적으로 polling해서 사용 가능한 Connection이 생겼는지 감지하는 단계로 들어가게 된다
- timeout이 10ms보다 적게 남아있으면 polling을 중단하고 return null

<div style="text-align: left">
  <img src="/assets/img/posts/2023-11-23-HikariCP%202%20-%20Connection을%20다루는%20메커니즘/img25.png" alt="img"/>
</div>

- Connection을 반납하는 과정에서 handoffQueue에 넣어주고 이 때 handoffQueue를 polling하고 있던 Thread는 반납된 Connection을 얻어서 사용하게 된다

> Timeout이 유효한 시간동안 <span style="color:red">handoffQueue를 polling하면서 사용 가능한 Connection이 생겼는지 감지</span>하고 생겼다면 해당 Connection을 얻게 된다
{: .prompt-info }

<br>

###### **_<span style="color:green">④ 못얻음 = return null</span>_**

<div style="text-align: left">
  <img src="/assets/img/posts/2023-11-23-HikariCP%202%20-%20Connection을%20다루는%20메커니즘/img26.png" alt="img"/>
</div>

- threadList → sharedList → handoffQueue를 거쳐도 못얻었으면 timeout + return null과 함께 예외가 발생한다

<br>

일단 여기까지 ConcurrentBag의 borrow()를 통해서 Connection을 얻는 프로세스를 정리해보자

> 1. threadList(이전에 사용했던 Connection) 탐색하면서 사용 가능한 Connection 조회<br>
> 2. sharedList(전체 Connection) 탐색하면서 사용 가능한 Connection 조회<br>
> 3. handoffQueue에서 대기 + polling하면서 사용 가능한 Connection 감지
{: .prompt-tip }

##### 2-2) validation

ConcurrentBag의 borrow()를 통해서 사용 가능한 Connection을 얻었으면 바로 사용해도 될까?

- No

<div style="text-align: left">
  <img src="/assets/img/posts/2023-11-23-HikariCP%202%20-%20Connection을%20다루는%20메커니즘/img27.png" alt="img"/>
</div>

- HikariCP에서 Connection을 생성할때는 maxLifeTime & keepAliveTime 설정을 고려해서 진짜 쓸 수 있는지 마킹을 진행하게 된다

<div style="text-align: left">
  <img src="/assets/img/posts/2023-11-23-HikariCP%202%20-%20Connection을%20다루는%20메커니즘/img28.png" alt="img"/>
</div>

- 유효하지 않은 Connection임을 validation했다면 softEvictConnection을 진행한다

<div style="text-align: left">
  <img src="/assets/img/posts/2023-11-23-HikariCP%202%20-%20Connection을%20다루는%20메커니즘/img29.png" alt="img"/>
</div>
<div style="text-align: left">
  <img src="/assets/img/posts/2023-11-23-HikariCP%202%20-%20Connection을%20다루는%20메커니즘/img30.png" alt="img"/>
</div>
<div style="text-align: left">
  <img src="/assets/img/posts/2023-11-23-HikariCP%202%20-%20Connection을%20다루는%20메커니즘/img31.png" alt="img"/>
</div>
<div style="text-align: left">
  <img src="/assets/img/posts/2023-11-23-HikariCP%202%20-%20Connection을%20다루는%20메커니즘/img32.png" alt="img"/>
</div>

1. 해당 Connection에 Evict 마크를 새긴다
2. 소유권(owner)이 있거나 Connection의 상대가 STATE_NOT_IN_USE면 Connection을 close한다 
   - STATE_RESERVED로 상태를 바꾸는데 이 상태는 해당 Connection이 종료되어야 한다는 것을 의미
3. try-with-resource를 통해서 Connection을 release

<div style="text-align: left">
  <img src="/assets/img/posts/2023-11-23-HikariCP%202%20-%20Connection을%20다루는%20메커니즘/img33.png" alt="img"/>
</div>

다시 돌아와서 ConcurrentBag#borrow()를 통해서 Connection을 획득하면 위에서 말한 validation을 진행한다

1. Evict 마크 O 
   - 가져온 Connection은 사용할 수 없는 Connection이고 close
2. Evict 마크 X 
   - 현재로부터 마지막 Access가 aliveBypassWindowMs보다 큰지 + Connection이 Dead 상태인지 확인한 후 모두 맞다면 close 
   - Dead 상태 판단 Query는 SELECT 1을 통해서 확인한다

<br>
> 얻은 Connection에 대한 추가 검증 끝에 poolEntry.createProxyConnection을 통해서 최종적으로 HikariProxyConnection을 획득하게 된다

### 3. Connection 반납

HikariProxyConnection을 얻은 후 사용하다가 더이상 쓸 일이 없다면 반납을 하게 된다

#### 1) ProxyConnection#close()

<div style="text-align: left">
  <img src="/assets/img/posts/2023-11-23-HikariCP%202%20-%20Connection을%20다루는%20메커니즘/img34.png" alt="img"/>
</div>

#### 2) PoolEntry#recycle()

> 메소드 네이밍을 통해서 <span style="color:red">완전한 close가 아니라 재사용을 위해서 DBCP에 반납</span>하는 느낌이라는 것을 유추할 수 있다

<div style="text-align: left">
  <img src="/assets/img/posts/2023-11-23-HikariCP%202%20-%20Connection을%20다루는%20메커니즘/img35.png" alt="img"/>
</div>

- lastAccessed를 기록하고 recycle을 진행한다

#### 3) HikariPool#recycle()

<div style="text-align: left">
  <img src="/assets/img/posts/2023-11-23-HikariCP%202%20-%20Connection을%20다루는%20메커니즘/img36.png" alt="img"/>
</div>

- 메트릭 관련 지표를 수집하고 ConcurrentBag의 requite를 호출한다

#### 4) ConcurrentBag#requite()

<div style="text-align: left">
  <img src="/assets/img/posts/2023-11-23-HikariCP%202%20-%20Connection을%20다루는%20메커니즘/img37.png" alt="img"/>
</div>

1. PoolEntry[Connection]의 상태를 STATE_NOT_IN_USE로 변경한다
2. <span style="color:red">handoffQueue에서 polling을 통해서 Connection을 대기하는 waiter들이 존재</span>한다면 반납하는 Connection을 제공
3. waiter가 없다면 최대 50개까지 보관 가능한 threadList에 반납한 Connection을 넣어준다

<br>

## 주요 옵션

### 1. maximumPoolSize [Default = 10]

> Idle + InUse를 합친 DBCP에서 관리할 수 있는 최대 Connection
{: .prompt-info }

- maximumPoolSize에 도달한 후 Idle Connection 없이 전부 InUse 상태라면 Connection을 얻기 위해서 대기해야 한다 (getConnection)
- 이 때 `connectionTimeout[ms] 만큼 대기`한다

<br>

[HikariCP Pool Size Docs](https://github.com/brettwooldridge/HikariCP/wiki/About-Pool-Sizing)

- maximumPoolSize가 많다고 무조건 성능이 좋아질까?
- [이전 포스팅](https://sjiwon.github.io/posts/HikariCP-1-DBCP/)에서도 언급했듯이 ThreadPool이든 DBCP든 내부에서 관리하는 리소스가 많다고 해서 그에 비례적으로 성능도 좋아지는 것은 아니다

<br>

Nginx vs Apache를 예로 들어보자<br>
- Apache = Client Connection당 하나의 쓰레드가 처리하는 구조
- Nginx = `Event-Driven`기반으로 Client Connection들을 적은 프로세스/쓰레드를 활용해서 비동기적으로 처리

<br>
위와 같은 논리라면 자원이 많다고 비례해서 성능이 좋아지면 Apache가 Nginx보다 성능이 좋아야 하지 않을까?<br>
현실은 그렇지 않다
  - Apache
    - Client Connection당 하나의 쓰레드가 담당해서 처리하기 때문에 Connection이 많아질수록 `쓰레드 생성, CPU + 메모리 낭비, ..`가 심해진다
      - C10K Problem
    - 서버의 프로세스가 Blocking되면 처리가 완료될때까지 대기해야 한다
  - Nginx
    - 적은 양의 쓰레드 + Event-Driven 기반 비동기로 동작하기 때문에 적은 양의 쓰레드만 사용되고 `Context Switching, CPU, 메모리`를 효율적으로 사용한다
    - 모든 I/O를 Event Listener에게 Delegating하고 그에 따라서 흐름이 끊기지 않고 응답이 빠르게 진행된다

<br>
그리고 HikariCP 팀에서 자체적으로 [Performance Test](https://github.com/brettwooldridge/HikariCP/wiki/About-Pool-Sizing#but-why)를 진행한 결과 오히려 PoolSize가 줄어들었을 때 응답 속도가 개선됨을 확인하였다

> You can see from the video that reducing the connection pool size alone, in the absence of any other change, decreased the response times of the application from ~100ms to ~2ms -- over 50x improvement.

##### Pool Locking

> 하나의 Thread에서 Connection을 획득한 후 <span style="color:red">추가적인 Connection을 획득하려고 하는 순간 Pool의 Idle Connection이 고갈되어서 획득하지 못하고 대기하는 현상</span>

### 2. minimumIdle [Default = maximumPoolSize]

> DBCP에서 최소로 유지해야 할 Idle Connection 수
{: .prompt-info }

- HikariCP Docs에서는 성능 & 급증하는 요청에 대한 응답성을 확보하기 위해서 이 값을 건드리지 말고 <span style="color:red">maximumPoolSize와 동일하게 유지하라고 권장</span>한다

### 3. connectionTimeout [Default = 30000ms = 30s]

> Client가 DB Connection을 획득하려고 기다릴 수 있는 최대 시간
{: .prompt-info }

- 이 시간을 초과해서 기다린다면 SQLException이 발생하게 된다

### 4. idleTimeout [Default = 600000ms = 10m && Min = 10000ms = 10s]

> Pool에서 Idle 상태로 유지될 수 있는 최대 시간
{: .prompt-info }

- minimumIdle만큼의 Connection은 제외하고 나머지 Idle Connection에 대해서 적용되는 Timeout

<div style="text-align: left">
  <img src="/assets/img/posts/2023-11-23-HikariCP%202%20-%20Connection을%20다루는%20메커니즘/img38.png" alt="img"/>
</div>

idleTimeout은 minimumIdle < maximumPoolSize의 경우에 한해서 의미있는 옵션인데 이 부분은 당연해보인다

- Pool 입장에서는 <span style="color:red">minimumIdle만큼의 Connection은 반드시 유지</span>해야 된다
  - 그런데 minimumIdle == maximumPoolSize라면? idleTimeout이 지나간다고 하더라도 없애버리면 어차피 다시 채워야한다
  - 이럴거면 애초에 idleTimeout이 지나서 Connection을 close한다고 하더라도 다시 Connection을 만들어야 하는데 이 리소스를 소모할바에 그냥 유지하는게 당연히 낫다

### 5. maxLifeTime [Default = 1800000ms = 30m && Min = 30000ms = 30s]

> Connection의 최대 수명 시간
{: .prompt-info }

- InUse Connection은 절대로 제거되지 않고 Idle 상태로 close되어야 제거된다
