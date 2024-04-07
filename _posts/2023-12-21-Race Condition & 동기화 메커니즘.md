---
title: Race Condition & 동기화 메커니즘
date: 2023-12-21 10:00 +0900
aliases: null
tags:
  - OS
  - Race Condition
  - Critical Section
  - Context Switching
  - Busy Waiting
  - Block & Wake-Up
  - Mutex
  - Semaphore
  - Monitor Lock
  - 자바 synchronized
image: /assets/img/thumbnails/OS.png
categories:
  - Computer Science
  - OS
---

## Race Condition

멀티 쓰레드 프로그래밍을 하게 된다면 Race Condition이라는 말은 누구나 한번쯤은 듣는 단어이다<br>
그러면 Race Condition이란 무엇일까?

> 멀티 쓰레드 환경에서 <span style="color:red">공유 자원에 대한 Write Operation</span>을 진행할 때 순서나 여러 조건에 의해서 결과값에 영향을 줄 수 있는 상황
{: .prompt-tip }

<br>

여기서 결과값에 영향을 줄 수 있는 상황이란 다음과 같은 상황을 의미한다

```text
공유 자원 count = 10이 존재
-> ThreadA = count++ 작업 진행
-> ThreadB = count-- 작업 진행
```

|     |                 ThreadA                 |                ThreadB                 |
|:---:|:---------------------------------------:|:--------------------------------------:|
| (1) |         RegisterA = count (10)          |         RegisterB = count (10)         |
| (2) |     RegisterA = RegisterA + 1 (11)      |     RegisterB = RegisterB - 1 (9)      |
| (3) | ***<ins>count = RegisterA (10)</ins>*** |                                        |
| (4) |                                         | ***<ins>count = RegisterB (9)</ins>*** |

`ThreadA는 ++ Operation`을 진행하고 `ThreadB는 -- Operation`을 진행하므로 전체적인 결과값에는 변화가 없는게 정상일 것이다<br>
하지만 <ins>공유 자원 count에 대한 Race Condition이 발생</ins>하고 그에 따라서 결과값에 이상이 생긴것이다

- 프로그래밍 레벨에서의 count++, count--는 마치 하나의 연산처럼 보인다
- 하지만 로우 레벨로 들어가게 되면 `Load -> Increase/Decrease -> Store` 총 3단계를 거치게 된다

### 해결 조건

이러한 Race Condition을 해결하기 위해서는 어떤 조건들을 만족해야할까?

#### 1. Mutual Exclusion

> 특정 프로세스/쓰레드가 Critical Section에 들어가서 작업을 진행할 때 또 다른 프로세스/쓰레드는 절대로 들어가게 하면 안된다

Mutual Exclusion을 만족하도록 아키텍처를 구성하게 된다면 추가적인 2가지 문제가 발생할 수 있다

1. Deadlock
2. Starvation

#### 2. Progress

> Critical Section에 어떠한 프로세스/쓰레드도 존재하지 않는다면 대기중인 어떠한 프로세스/쓰레드들도 못들어가는 상황은 발생하면 안된다

- 비어있으면 누구라도 들어가서 진행해라

#### 3. Bounded Waiting

> 평생 Critical Section에 들어가지 못해서 무한 대기하는 상황은 발생하면 안된다

- 무한 대기하지 않도록 대기 시간을 제어해라

<br>
그런데 사실 이 3가지 조건을 모두 만족시키면서 설계하기는 굉장히 어렵고 까다롭다<br>
따라서 Critical Section이 발생하더라도 Deadlock이나 Starvation 문제를 어떻게 효율적으로 해결하냐를 고민하는게 더 현실적이다

<br>

## Race Condition에 대한 해결방안 1) Mutex

> <span style="color:red">Mut</span>ual <span style="color:red">Ex</span>clusion을 축약한 단어로써 말그대로 `상호배제`를 통해서 Critical Section에 대한 동기화를 진행하는 기법
{: .prompt-info }

- Critical Section에 들어가기 전에 반드시 Mutex Lock을 획득(acquire)해야 한다
- Critical Section에서 나오면 반드시 Mutex Lock을 반납(release)해야 한다

### Mutex 특징

- `Boolean 타입의 Lock 개념`을 활용해서 구현하기 때문에 동기화 대상은 오직 하나이다
- Mutex Lock을 획득하게 된다면 Lock은 온전히 소유하게 되고 그에 대한 해제도 반드시 책임져야 한다 
  - Mutex Lock을 획득한 프로세스/쓰레드만이 해제할 수 있다는 의미
- 프로세스/쓰레드 레벨의 범위를 가지고 있고 프로세스/쓰레드가 종료될 때 자동으로 Clean Up된다

### 간단한 구현 (pseudo-code)

```text
boolean availabie;
 
acquire() {
    while (!available) { // busy waiting... }
    available = true;
}
 
release() {
    available = true;
}
```

정말 간단하게 Mutex의 acquire & release에 대한 슈도 코드를 작성해보았다<br>
위의 방식은 <span style="color:red">Busy Waiting(Spin Lock)</span>이라는 문제가 발생하게 된다

> 한 프로세스/쓰레드가 Lock을 가지고 Critical Section으로 진입하게 되면, 다른 프로세스/쓰레드가 Lock을 획득하려고 시도할 때 <span style="color:red">지속적으로 Loop를 돌면서 Lock Available 상태를 감시</span>하는 과정

#### Busy Waiting의 문제점

##### ***1) CPU 낭비***

-  Lock이 해제되기를 기다리면서 CPU를 계속 점유함으로써 다른 유용한 작업을 할 수 없게 되어 자원을 낭비

##### ***2)️ Priority Inversion***

예시 시나리오를 살펴보자 (우선순위 높은 ThreadA & 우선순위 낮은 ThreadB)

1. ThreadB가 로직을 실행하면서 공유 자원에 접근하기 위해서 Mutex Lock을 획득했다
2. 이후 ThreadA도 로직을 실행하면서 동일한 Mutex Lock을 획득하려고 시도한다

여기서 Mutex Lock은 `획득한 프로세스/쓰레드만이 해제`할 수 있기 때문에 ThreadA가 우선순위가 높다고 하더라도 ThreadB가 해제할 때까지 대기해야 한다

- 따라서 이 과정에서 ThreadA는 ThreadB가 Mutex Lock을 해제할때까지 유용한 작업을 수행할 수 없고 CPU를 낭비하게 된다
- 특히 ThreadB는 우선순위가 낮기 때문에 만약 우선순위가 더 높은 ThreadC, D, ...가 개입하면 CPU를 다시 할당받기 위해서 오랜 시간이 걸릴 것이다

<br>
이러한 Priority Inversion 문제를 해결하기 위해서 `Priority Inheritance`를 활용해서 일시적으로 Step 2에서 ThreadB의 우선순위를 ThreadA와 동등하게 상승시킨 후 Mutex Lock 해제에 대한 빠른 진행을 유도시킬 수 있다

- 물론 작업이 완료되면 ThreadB는 원래 우선순위로 돌아간다

##### ***3) Starvation***

- 여러 쓰레드가 동시에 Mutex Lock을 경쟁할 때 일부 Thread가 다른 Thread들에 의해 계속해서 획득하지 못해서 영원히 Critical Section에 진입하지 못하는 상황

<br>
이렇게 CPU를 낭비하고 여러 문제가 발생할 수 있는 Busy Waiting 대신 Block & Wake-Up 방식을 활용할 수도 있다

> Lock을 얻기 위해서 무한 루프를 도는것이 아니라 <span style="color:red">Queue에서 대기하다가 Lock 해제에 대한 Notify를 받으면 그때 다시 경합</span>하는 과정

1. CPU를 의미없이 감시하는 과정에 활용하지 않고 Lock을 대기하는 상황에서는 CPU에 대한 점유를 해제하고 다른 쓰레드가 의미있게 활용할 수 있도록 할 수 있다
2. CPU를 점유하면서 대기하지 않기 때문에 Priority Inversion 문제 또한 해결할 수 있다

<br>
하지만 Lock 경쟁이 적거나 대기시간이 짧을 경우 오히려 Busy Waiting 방식을 활용하는게 더 효율적일 수 있다

- Context Switching에 대한 오버헤드가 더 심각한 상황

<br>

## Race Condition에 대한 해결방안 2) Semaphore

> 공유 자원에 접근할 수 있는 쓰레드/프로세스의 수에 제한을 두는 방식
{: .prompt-info }

위에서 설명한 Mutex는 Boolean 타입의 변수를 통해서 오직 하나의 동기화 대상을 가지고 있었다<br>
반면에 Semaphore는 <span style="color:red">1개 이상의 동기화 대상</span>을 가지고 있는 기법이다

<br>
자바에서는 Semaphore API를 제공해주고 있다

<div style="text-align: left">
  <img src="/assets/img/posts/2023-12-21-Race%20Condition%20&%20동기화%20메커니즘/img1.png" alt="img"/>
</div>

- int permits = Semaphore 공유 자원의 수

<div style="text-align: left">
  <img src="/assets/img/posts/2023-12-21-Race%20Condition%20&%20동기화%20메커니즘/img2.png" alt="img"/>
</div>
<div style="text-align: left">
  <img src="/assets/img/posts/2023-12-21-Race%20Condition%20&%20동기화%20메커니즘/img3.png" alt="img"/>
</div>


### Semaphore 테스트

Semaphore API를 활용해서 Race Condition 상황에서 Semaphore 유무에 따라 결과가 어떻게 달라지는지 확인해보자

```java
public class SharedResource {
    private int count = 0;

    public void increase() {
        count++;
        System.out.printf("%s increased count to %d\n", Thread.currentThread().getName(), count);
    }
}
```

#### 1. Semaphore X

```java
public class PureLogic implements Runnable {
    private final SharedResource sharedResource;

    public PureLogic(final SharedResource sharedResource) {
        this.sharedResource = sharedResource;
    }

    @Override
    public void run() {
        for (int i = 0; i < 5; i++) {
            try {
                Thread.sleep(100);
            } catch (final InterruptedException e) {
                throw new RuntimeException(e);
            }
            sharedResource.increase();
        }
    }
}

final PureLogic logic = new PureLogic(new SharedResource());
final Thread threadA = new Thread(logic);
final Thread threadB = new Thread(logic);

threadA.start();
threadB.start();
```

<div style="text-align: left">
  <img src="/assets/img/posts/2023-12-21-Race%20Condition%20&%20동기화%20메커니즘/img4.png" alt="img"/>
</div>

- Race Condition에 의한 데이터 정합성에 문제가 발생하였다

#### 2. Semaphore O (Binary Semaphore)

```java
public class BinarySemaphore {
    private final Semaphore semaphore = new Semaphore(1);

    public void acquire() {
        try {
            semaphore.acquire();
            System.out.printf("[%s] Semaphore 획득...\n", Thread.currentThread().getName());
        } catch (final InterruptedException e) {
            throw new RuntimeException(e);
        }
    }

    public void release() {
        semaphore.release();
        System.out.printf("[%s] Semaphore 반납...\n", Thread.currentThread().getName());
    }
}

public class LogicWithBinarySemaphore implements Runnable {
    private final BinarySemaphore binarySemaphore;
    private final SharedResource sharedResource;

    public LogicWithBinarySemaphore(
            final BinarySemaphore binarySemaphore,
            final SharedResource sharedResource
    ) {
        this.binarySemaphore = binarySemaphore;
        this.sharedResource = sharedResource;
    }

    @Override
    public void run() {
        try {
            binarySemaphore.acquire();

            for (int i = 0; i < 5; i++) {
                try {
                    Thread.sleep(100);
                } catch (final InterruptedException e) {
                    throw new RuntimeException(e);
                }
                sharedResource.increase();
            }
        } finally {
            binarySemaphore.release();
        }
    }
}

final LogicWithBinarySemaphore logic = new LogicWithBinarySemaphore(
        new BinarySemaphore(),
        new SharedResource()
);
final Thread threadA = new Thread(logic);
final Thread threadB = new Thread(logic);

threadA.start();
threadB.start();
```

<div style="text-align: left">
  <img src="/assets/img/posts/2023-12-21-Race%20Condition%20&%20동기화%20메커니즘/img5.png" alt="img"/>
</div>

- `Binary Semaphore`를 활용해서 Critical Section에 대한 Mutual Exclusion을 보장할 수 있다

#### 3. Semaphore O (Count Semaphore)

```java
public class CountSemaphore {
    private final Semaphore semaphore = new Semaphore(2);

    public void acquire() {
        try {
            semaphore.acquire();
            System.out.printf("[%s] Semaphore 획득...\n", Thread.currentThread().getName());
        } catch (final InterruptedException e) {
            throw new RuntimeException(e);
        }
    }

    public void release() {
        semaphore.release();
        System.out.printf("[%s] Semaphore 반납...\n", Thread.currentThread().getName());
    }
}

public class LogicWithCountSemaphore implements Runnable {
    private final CountSemaphore countSemaphore;
    private final SharedResource sharedResource;

    public LogicWithCountSemaphore(
            final CountSemaphore countSemaphore,
            final SharedResource sharedResource
    ) {
        this.countSemaphore = countSemaphore;
        this.sharedResource = sharedResource;
    }

    @Override
    public void run() {
        try {
            countSemaphore.acquire();

            for (int i = 0; i < 5; i++) {
                try {
                    Thread.sleep(100);
                } catch (final InterruptedException e) {
                    throw new RuntimeException(e);
                }
                sharedResource.increase();
            }
        } finally {
            countSemaphore.release();
        }
    }
}

final LogicWithCountSemaphore logic = new LogicWithCountSemaphore(
        new CountSemaphore(),
        new SharedResource()
);
final Thread threadA = new Thread(logic);
final Thread threadB = new Thread(logic);

threadA.start();
threadB.start();
```

<div style="text-align: left">
  <img src="/assets/img/posts/2023-12-21-Race%20Condition%20&%20동기화%20메커니즘/img6.png" alt="img"/>
</div>

- Count Semaphore(2)로 적용하였더니 다시 데이터 정합성에 문제가 발생하였다

Count Semaphore의 경우 <span style="color:red">공유 자원에 대해서 N개의 프로세스/쓰레드의 동시 접근</span>을 허용한다<br>
따라서 이러한 구조에서 `공유 자원에 대한 Write`가 발생할 경우 데이터 정합성은 깨질 수 밖에 없다

### vs Mutex

#### 동기화 대상

- Mutex는 동기화 대상이 오직 1개이다
- Semaphore는 동기화 대상이 1개 이상이다

#### Lock에 대한 소유권

- Mutex는 Lock을 소유한 프로세스/쓰레드만이 Lock을 반납할 수 있다
- Semaphore는 Lock을 소유하고 있지 않더라도 Lock을 반납할 수 있다

<br>

## Race Condition에 대한 해결방안 3) Monitor Lock

위에서 살펴본 Mutex & Semaphore는 Critical Section 문제에 대한 해결책이 될 수 있지만 Human Error가 발생할 여지가 존재한다

- Correctness에 대한 입증이 어렵고 한번의 휴먼 에러로 전체 시스템에 치명적인 영향을 줄 수 있다

```text
// 정상
wait(S);
...
signal(S);
 
// 오류 - 1
signal(S);
...
wait(S);
 
// 오류 - 2
wait(S);
...
wait(S);
```

이러한 휴먼 에러와 같은 문제를 보완하기 위해서 Monitor가 등장하게 되었다

> 프로세스 동기화에 대해서 굉장히 편리하고 효율적인 메커니즘을 제공해주는 ADT
{: .prompt-info }

<div style="text-align: left">
  <img src="/assets/img/posts/2023-12-21-Race%20Condition%20&%20동기화%20메커니즘/img7.png" alt="img"/>
</div>

- Monitor 내부에는 다음과 같은 자원들이 존재한다 
  - 공유 데이터
  - Monitor Lock
  - 1개 이상의 Condition Variable
- Monitor 내부의 공유 자원에 접근하기 위해서는 ADT에서 제공해주는 API를 통해서만 접근이 가능하다
- Monitor 내부에서는 <span style="color:red">오직 하나의 프로세스/쓰레드만이 Active</span>하다

### Entry Queue & Waiting Queue

#### Entry Queue

> Monitor 진입 시점에서 Mutual Exclusion을 보장하기 위한 Queue<br>
> -> <span style="color:green">Mutual Exclusion</span>을 위한 Queue

- Entry Queue를 통해서 Monitor 내부에 동시에 여러 프로세스/쓰레드들이 진입하는 것을 막는다

#### Waiting Queue

> 내부에서 여러 로직 실행중에 Condition Variable의 wait()에 의해서 잠시 쉬러 들어가는 Queue<br>
> -> <span style="color:green">Conditional Synchronization</span>을 위한 Queue

- Active상태에서 로직을 실행하다가 Blocking되는 순간 들어가는 Queue
- Active Process/Thread가 signal()을 호출하면 Waiting Queue에 존재하는 여러 프로세스/쓰레드들이 Active를 위해서 다시 경쟁하게 된다

### 자바에서 제공해주는 synchronized

> 자바에서 제공해주는 `synchronized 키워드`는 이러한 Monitor 기반의 동기화 메커니즘을 제공한다
{: .prompt-info }

#### 1. synchronized method

> Method 단위에 synchronized를 적용하게 되면 ***<ins>인스턴스 단위 Monitor Lock</ins>***이 걸리게 된다

```java
public class Sync {
    public void a() {
        execute("void a()");
    }

    public synchronized void b() {
        execute("synchronized void b()");
    }

    public synchronized void c() {
        execute("synchronized void c()");
    }

    private static void execute(String title) {
        try {
            System.out.printf("%s: (%s) START\n", Thread.currentThread().getName(), title);

            Thread.sleep(500);
            for (int i = 0; i < 2; i++) {
                System.out.printf("\t%s running...\n", Thread.currentThread().getName());
            }
        } catch (InterruptedException ignored) {
        } finally {
            System.out.printf("%s: (%s) END\n", Thread.currentThread().getName(), title);
        }
    }
}
```

<div style="text-align: left">
  <img src="/assets/img/posts/2023-12-21-Race%20Condition%20&%20동기화%20메커니즘/img8.png" alt="img"/>
</div>

#### 2. synchronized block

> Block 단위에 synchronized를 적용하게 되면 Block의 type에 따라 ***<ins>인스턴스/객체/클래스 단위 Monitor Lock</ins>***이 걸리게 된다<br>

- `synchronized (this)` = Method 단위 synchronized와 동일하게 동작
- `synchronized (객체)` = 해당 객체에 Lock을 거는 Block간에 Monitor Lock을 공유
- `synchronized (클래스.class)` = 해당 클래스에 Lock을 거는 Block간에 Monitor Lock을 공유

```java
public class SyncBlock {
    private static final Object lock = new Object();

    static class LockObject {
    }

    public void a1() {
        synchronized (this) {
            execute("void a1() :: synchronized (this)");
        }
    }

    public void a2() {
        synchronized (this) {
            execute("void a2() :: synchronized (this)");
        }
    }

    public void b1() {
        synchronized (lock) {
            execute("void b1() :: synchronized (lock)");
        }
    }

    public void b2() {
        synchronized (lock) {
            execute("void b2() :: synchronized (lock)");
        }
    }

    public void c1() {
        synchronized (LockObject.class) {
            execute("void c1() :: synchronized (LockObject.class)");
        }
    }

    public void c2() {
        synchronized (LockObject.class) {
            execute("void c2() :: synchronized (LockObject.class)");
        }
    }

    private static void execute(String title) {
        try {
            System.out.printf("%s: (%s) START\n", Thread.currentThread().getName(), title);

            Thread.sleep(500);
            for (int i = 0; i < 2; i++) {
                System.out.printf("\t%s running...\n", Thread.currentThread().getName());
            }
        } catch (InterruptedException ignored) {
        } finally {
            System.out.printf("%s: (%s) END\n", Thread.currentThread().getName(), title);
        }
    }
}
```

<div style="text-align: left">
  <img src="/assets/img/posts/2023-12-21-Race%20Condition%20&%20동기화%20메커니즘/img9.png" alt="img"/>
</div>

#### 3. static synchronized method

> ***<ins>클래스 단위 Monitor Lock</ins>***이 걸리게 된다

- static synchronized method & synchronized method간에 Monitor Lock은 공유되지 않고 개별적으로 관리된다

```java
public class StaticSync {
    public void a() {
        execute("void a()");
    }

    public synchronized void b() {
        execute("synchronized void b()");
    }

    public static synchronized void c1() {
        execute("static synchronized void c1()");
    }

    public static synchronized void c2() {
        execute("static synchronized void c2()");
    }

    private static void execute(String title) {
        try {
            System.out.printf("%s: (%s) START\n", Thread.currentThread().getName(), title);

            Thread.sleep(500);
            for (int i = 0; i < 2; i++) {
                System.out.printf("\t%s running...\n", Thread.currentThread().getName());
            }
        } catch (InterruptedException ignored) {
        } finally {
            System.out.printf("%s: (%s) END\n", Thread.currentThread().getName(), title);
        }
    }
}
```

<div style="text-align: left">
  <img src="/assets/img/posts/2023-12-21-Race%20Condition%20&%20동기화%20메커니즘/img10.png" alt="img"/>
</div>

#### 4. static synchronized block

> static synchronized block에 들어갈 수 있는 인자는 ***<ins>정적 인스턴스 & 클래스 타입만 허용</ins>***한다

```java
public class StaticSyncBlock {
    private static final Object lock = new Object();

    static class LockObject {
    }

    public static void a1() {
        synchronized (lock) {
            execute("static void a1() :: synchronized (lock)");
        }
    }

    public static void a2() {
        synchronized (lock) {
            execute("static void a2() :: synchronized (lock)");
        }
    }

    public static void b1() {
        synchronized (LockObject.class) {
            execute("static void b1() :: synchronized (LockObject.class)");
        }
    }

    public static void b2() {
        synchronized (LockObject.class) {
            execute("static void b2() :: synchronized (LockObject.class)");
        }
    }

    private static void execute(String title) {
        try {
            System.out.printf("%s: (%s) START\n", Thread.currentThread().getName(), title);

            Thread.sleep(500);
            for (int i = 0; i < 2; i++) {
                System.out.printf("\t%s running...\n", Thread.currentThread().getName());
            }
        } catch (InterruptedException ignored) {
        } finally {
            System.out.printf("%s: (%s) END\n", Thread.currentThread().getName(), title);
        }
    }
}
```

<div style="text-align: left">
  <img src="/assets/img/posts/2023-12-21-Race%20Condition%20&%20동기화%20메커니즘/img11.png" alt="img"/>
</div>
