---
title: Transaction 동기화
date: 2023-01-09 13:00 +0900
aliases: null
tags: [ Spring, Transaction, Connection, PlatformTransactionManager, TransactionSynchronizationManager ]
image: /assets/img/thumbnails/Spring.png
categories: [ Skill, Spring ]
---

## Transaction - Connection간의 관계

트랜잭션을 열고 유지하기 위해서 트랜잭션의 `시작 ~ 끝`까지 다음과 같은 이유로 DB Connection을 동기화해야 한다

- 트랜잭션 내부의 여러 연산 로직은 `최종적으로 원자성 (All or Nothing)`을 보장해야 한다
- 이 과정에서 모든 연산은 동일한 DB Connection을 사용해야 한다
- 만약 중간에 Connection이 변경되면 연산 간의 데이터 일관성을 보장할 수 없게 된다

### 파라미터를 통한 Connection 동기화 방법

```kotlin
fun logic() {
    val connection: Connection = ...
    componentA.execute(connection, ...)
    componentB.execute(connection, ...)
    componentC.execute(connection, ...)
    ...
}
```

가장 쉽게 생각할 수 있는 방법은 트랜잭션 연산간에 Connection을 파라미터로 넘겨줌으로써 동기화하는 방식이다<br>
하지만 이 방식은 좀만 생각해보면 굉장히 귀찮고 휴먼 에러가 발생할 여지가 많은 방식이다<br>
그리고 전역에 걸친 트랜잭션 관리가 굉장히 까다롭다면 구현 자체가 복잡해질 여지가 있다

Spring에서는 이러한 문제를 어떻게 해결하고 있을까?

<br>

## TransactionSynchronizationManager

> 동일한 Transaction간에 여러 리소스 동기화를 위해서 `임시 보관`해주는 컴포넌트

<div style="text-align: left">
  <img src="/assets/img/posts/2023-01-09-Transaction%20동기화/img1.png" alt="img"/>
</div>

- `ThreadLocal`을 통해서 쓰레드별로 트랜잭션에 필요한 여러 리소스를 동기화시켜준다

### PlatformTransactionManager + TransactionSynchronizationManager (With JPA)

#### 1. getTransaction

<div style="text-align: left">
  <img src="/assets/img/posts/2023-01-09-Transaction%20동기화/img2.png" alt="img"/>
</div>

- PlatformTransactionManager의 `getTransaction`은 새로운 트랜잭션을 생성하거나 기존 트랜잭션에 참여하는 로직을 담고 있다
- 이 과정에서 `doGetTransaction()`을 호출한다

#### 2. doGetTransaction &rarr; getResource

<div style="text-align: left">
  <img src="/assets/img/posts/2023-01-09-Transaction%20동기화/img3.png" alt="img"/>
</div>

- `doGetTransaction`의 내부에서는 `해당 트랜잭션과 관련된 리소스`들을 TransactionSynchronizationManager에서 가져온다
- 위의 케이스에서는 `JPA`를 활용하고 있기 때문에 `JpaTransactionManager`에 의해서 관련된 Resource들을 가져온다
  - EntityManager
  - Connection

#### 3. 트랜잭션 진행...

<div style="text-align: left">
  <img src="/assets/img/posts/2023-01-09-Transaction%20동기화/img4.png" alt="img"/>
</div>

- 리소스를 얻은 후 `Transaction Propagation`에 따라 이후 로직들을 진행한다

<br>

## Connection 관리 (반납 & 제거)

<div style="text-align: left">
  <img src="/assets/img/posts/2023-01-09-Transaction%20동기화/img5.png" alt="img"/>
</div>

logicA와 logicB가 하나의 트랜잭션 내부적으로 동일한 Connection을 통해서 DB와 상호작용을 한다고 가정하자

- call logicA (Propagation = REQUIRED)
- call logicB (Propagation = REQUIRED)

logicA의 로직이 모두 완료되었다고 하더라도 묶여있는 트랜잭션 자체가 종료된것은 아니기 때문에 `logicA가 사용한 Connection은 다시 TransactionSynchronizationManager에 반납`된다

logicB는 Propagation REQUIRED이므로 logicA가 새로 생성한 트랜잭션에 참여한다<br>
그리고 ***TransactionSynchronizationManager에서 동기화된 Connection***을 얻어서 로직을 진행한다

이 후 logicB가 종료된다면 이제서야 묶여있는 트랜잭션이 종료된다

최종적으로 트랜잭션이 commit/rollback에 의해 종료되는 시점에 TransactionSynchronizationManager에 있는 해당 Transaction의 Connection도 Connection Pool에 Release된다

```java
// AbstractPlatformTransactionManager
@Override
public final void commit(TransactionStatus status) throws TransactionException {
  ...

  DefaultTransactionStatus defStatus = (DefaultTransactionStatus) status;
  if (defStatus.isLocalRollbackOnly()) {
    ...
    processRollback(defStatus, false);
    return;
  }

  if (!shouldCommitOnGlobalRollbackOnly() && defStatus.isGlobalRollbackOnly()) {
    ...
    processRollback(defStatus, true);
    return;
  }

  processCommit(defStatus);
}

@Override
public final void rollback(TransactionStatus status) throws TransactionException {
  if (status.isCompleted()) {
    throw new IllegalTransactionStateException(
        "Transaction is already completed - do not call commit or rollback more than once per transaction");
  }

  DefaultTransactionStatus defStatus = (DefaultTransactionStatus) status;
  processRollback(defStatus, false);
}
```

- commit과 rollback에 대한 구현 코드를 살펴보면 `processRollback & processCommit`을 호출하는 것을 볼 수 있다

```java
// AbstractPlatformTransactionManager
private void processCommit(DefaultTransactionStatus status) throws TransactionException {
  try {
    ...
  } 
  finally {
    cleanupAfterCompletion(status);
  }
}

private void processRollback(DefaultTransactionStatus status, boolean unexpected) {
  try {
    ...
  }
  finally {
    cleanupAfterCompletion(status);
  }
}
```

- 두 로직 모두 finally 부분에서 `cleanupAfterCompletion(status)`를 호출한다

<div style="text-align: left">
  <img src="/assets/img/posts/2023-01-09-Transaction%20동기화/img6.png" alt="img"/>
</div>
