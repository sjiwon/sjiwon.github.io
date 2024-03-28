---
title: Transaction 전파
date: 2023-01-09 16:00 +0900
aliases: null
tags: [ Spring, Transaction, Connection, PlatformTransactionManager, TransactionSynchronizationManager ]
image: /assets/img/thumbnails/Spring.png
categories: [ Skill, Spring ]
---

## Transaction 전파

트랜잭션은 `시작 지점 & 종료 지점`이 명확하게 존재한다<br>
시작 지점은 `하나`이지만 종료 지점은 `둘`로 분류할 수 있다

1. Commit
2. Rollback

특정 로직을 진행하다가 추가적인 트랜잭션이 진입하는 경우를 생각해보자<br>
이 때 기존에 존재하는 트랜잭션에 대해서 추가적으로 진행되는 트랜잭션의 행동은 `트랜잭션 전파 속성`에 의해 결정된다

### 물리 트랜잭션 vs 논리 트랜잭션

단일 트랜잭션인 상황에서는 사실 트랜잭션의 개념 자체를 물리/논리로 나눌 필요가 없다

- 논리 트랜잭션 그 자체가 물리 트랜잭션이기 때문에

하지만 여러 트랜잭션이 중첩된 상황에서는 물리/논리라는 개념으로 분리될 필요가 있다

<div style="text-align: left">
  <img src="/assets/img/posts/2023-01-09-Transaction%20전파/img1.png" alt="img"/>
</div>

이러한 상황에서 물리 트랜잭션 / 논리 트랜잭션간에는 다음과 같은 규칙이 존재한다

> 1. 모든 논리 트랜잭션이 commit되어야 물리 트랜잭션이 commit된다<br>
> 2. 하나의 논리 트랜잭션이라도 rollback된다면 물리 트랜잭션은 rollback된다
{: .prompt-tip }

하지만 다음과 같은 상황에서는 약간 다르다

<div style="text-align: left">
  <img src="/assets/img/posts/2023-01-09-Transaction%20전파/img2.png" alt="img"/>
</div>

이 경우 <ins>***논리 트랜잭션 2는 전파 속성이 REQUIRES_NEW***</ins>이므로 물리 트랜잭션1과는 아예 관련이 없는 `새로운 트랜잭션`에서 동작한다<br>
따라서 위의 규칙은 예외없이 따라야 하지만 논리 트랜잭션 2의 경우 물리 트랜잭션 1에 어떠한 영향을 미치지 않는다

- 별개의 트랜잭션이므로

<br>
물리 트랜잭션과 논리 트랜잭션에 대해서 간단하게 정리 해보면 다음과 같다

`물리 트랜잭션`

- 실제 DB에 적용이 되는 트랜잭션 단위
- Connection을 통해서 commit-rollback을 하는 단위

`논리 트랜잭션`

- TransactionManager를 통해서 관리되는 트랜잭션 단위

### Spring에서의 기본 트랜잭션 처리 전략

Spring에서 `기본적으로 적용되는 예외`에 대한 트랜잭션 처리 전략은 다음과 같다

- Checked Exception = 커밋
- Unchecked Exception = 롤백

<br>
Checked냐 Unchecked냐에 따라 커밋/롤백되는 이 기본 전략은 어디까지나 Spring에서 `기본적으로 제공`해주는 전략이다
이를 오해하고 자바에서 예외 처리에 대한 기본 전략을 아래와 같이 이해하면 안된다

> 자바에서도 마찬가지로 Checked Exception = 커밋 & Unchecked Exception = 롤백?
{: .prompt-danger }

- 자바에서의 전략은 전적으로 개발자가 알아서 구현해야 한다
- 언어 레벨에서 위와 같은 메커니즘을 제공해주지 않는다

<br>
Spring에서도 원한다면 @Transactional의 `rollbackFor, noRollbackFor,...`등의 옵션으로 예외에 대한 처리 전략을 커스텀하게 적용할 수 있다

<br>

## Propagation에 따른 트랜잭션 처리 흐름

테스트 Commit/Rollback 구조
- Commit = 정상 흐름
- Rollback = Unchecked Exception 발생

```yaml
logging:
  level:
    org.springframework.orm.jpa.JpaTransactionManager: DEBUG
    org.hibernate.resource.transaction: DEBUG
```

### REQUIRED (기본값)

> 기존 트랜잭션 X: 새로운 트랜잭션 생성<br>
> 기존 트랜잭션 O: 기존 트랜잭션에 참여
{: .prompt-info }

#### 1) Main(Commit) & Sub(Commit)

```kotlin
// RequiredMainComponent
@Transactional
fun case1() {
    log.info("===== Main(REQUIRED) BEGIN =====")
    log.info(">> Main 로직 진행...")
    sub.commit()
    log.info("===== Main(REQUIRED) COMMIT =====")
}

// RequiredSubComponent
@Transactional
fun commit() {
    log.info("===== Sub(REQUIRED) BEGIN =====")
    log.info(">> Sub 로직 진행...")
    log.info("===== Sub(REQUIRED) COMMIT =====")
}

// Test
@Test
fun `Main(Commit) & Sub(Commit)`() {
    shouldNotThrowAny { main.case1() }
}
```

<div style="text-align: left">
  <img src="/assets/img/posts/2023-01-09-Transaction%20전파/required1.png" alt="img"/>
</div>

- 두 트랜잭션(물리, 논리) 모두 Commit이므로 최종 처리 역시 Commit으로 진행된다

#### 2) Main(Commit) & Sub(Rollback)

```kotlin
// RequiredMainComponent
@Transactional
fun case2() {
    log.info("===== Main(REQUIRED) BEGIN =====")
    log.info(">> Main 로직 진행...")
    try {
        sub.rollback()
    } catch (_: Exception) {
        log.info(">> 논리 트랜잭션 Exception 발생...")
    } finally {
        log.info("===== Main(REQUIRED) COMMIT =====")
    }
}

// RequiredSubComponent
@Transactional
fun rollback() {
    log.info("===== Sub(REQUIRED) BEGIN =====")
    log.info(">> Sub 로직 진행...")
    log.info("===== Sub(REQUIRED) ROLLBACK =====")

    throw TxException()
}

// Test
@Test
fun `Main(Commit) & Sub(Rollback)`() {
    shouldThrow<UnexpectedRollbackException> { main.case2() }
}
```

<div style="text-align: left">
  <img src="/assets/img/posts/2023-01-09-Transaction%20전파/required2.png" alt="img"/>
</div>

- `RequiredSubComponent` 논리 트랜잭션 내부에서 Unchecked Exception이 발생했고 그에 따라서 참여하고 있는 트랜잭션에 `rollbackOnly 마킹`을 하게 된다
- 그 후 `RequiredMainComponent` 물리 트랜잭션에서 Commit을 시도하려고 해도 `이미 rollbackOnly 마킹`이 되어있으므로 `UnexpectedRollbackException`과 함께 Rollback된다

#### 3) Main(Rollback) & Sub(Commit)

```kotlin
// RequiredMainComponent
@Transactional
fun case3() {
    log.info("===== Main(REQUIRED) BEGIN =====")
    log.info(">> Main 로직 진행...")
    sub.commit()
    log.info("===== Main(REQUIRED) ROLLBACK =====")

    throw TxException()
}

// RequiredSubComponent
@Transactional
fun commit() {
    log.info("===== Sub(REQUIRED) BEGIN =====")
    log.info(">> Sub 로직 진행...")
    log.info("===== Sub(REQUIRED) COMMIT =====")
}

// Test
@Test
fun `Main(Rollback) & Sub(Commit)`() {
    shouldThrow<TxException> { main.case3() }
}
```

<div style="text-align: left">
  <img src="/assets/img/posts/2023-01-09-Transaction%20전파/required3.png" alt="img"/>
</div>

- `RequiredSubComponent` 논리 트랜잭션은 정상적으로 진행되었지만 `RequiredMainComponent` 물리 트랜잭션에서 Unchecked Exception이 발생했기 때문에 Rollback을 진행한다

<br>

### REQUIRES_NEW

> 기존 트랜잭션 X: 새로운 트랜잭션 생성<br>
> 기존 트랜잭션 O: 새로운 트랜잭션 생성
{: .prompt-info }

- 기존 트랜잭션이 있든 없든 무조건 새로운 트랜잭션을 생성해서 진행한다
- 만약 기존 트랜잭션이 존재하는 경우 기존 트랜잭션을 `잠시 중단`하고 새로운 트랜잭션을 진행한다

#### 1) SubA(Commit) & SubB(Rollback)

```kotlin
// RequiresNewMainComponent
@Transactional
fun case1() {
    log.info("===== Main(REQUIRED) BEGIN =====")
    log.info(">> Main 로직 진행...")
    subA.commit()
    try {
        subB.rollback()
    } catch (_: Exception) {
    }
    log.info("===== Main(REQUIRED) COMMIT =====")
}

// RequiresNewSubComponentA
@Transactional
fun commit() {
    log.info("===== SubA(REQUIRED) BEGIN =====")
    log.info(">> SubA 로직 진행...")
    log.info("===== SubA(REQUIRED) COMMIT =====")
}

// RequiresNewSubComponentB
@Transactional(propagation = Propagation.REQUIRES_NEW)
fun rollback() {
    log.info("===== SubB(REQUIRES_NEW) BEGIN =====")
    log.info(">> SubB 로직 진행...")
    log.info("===== SubB(REQUIRES_NEW) ROLLBACK =====")

    throw TxException()
}

// Test
@Test
fun `SubA(Commit) & SubB(Rollback)`() {
    shouldNotThrowAny { main.case1() }
}
```

<div style="text-align: left">
  <img src="/assets/img/posts/2023-01-09-Transaction%20전파/requiresnew1.png" alt="img"/>
</div>

- Main & SubA는 정상적으로 Commit, SubB는 Unchecked Exception이 발생하였다
- 여기서 SubB는 `REQUIRES_NEW`이므로 Main & SubA와는 별개의 트랜잭션에서 로직이 진행된다
  - Main & SubA의 기존 트랜잭션은 `잠시 중단 (suspending)`된다
- 따라서 결과는 다음과 같이 진행된다
  - Main & SubA의 TX1 = Commit
  - SubB의 TX2 = Unchecked Exception으로 인한 Rollback

#### 2) SubA(Rollback) & SubB(Rollback)

```kotlin
// RequiresNewMainComponent
@Transactional
fun case2() {
    log.info("===== Main(REQUIRED) BEGIN =====")
    log.info(">> Main 로직 진행...")
    try {
        subA.rollback()
    } catch (_: Exception) {
    }
    subB.commit()
    log.info("===== Main(REQUIRED) COMMIT =====")
}

// RequiresNewSubComponentA
@Transactional
fun rollback() {
    log.info("===== SubA(REQUIRED) BEGIN =====")
    log.info(">> SubA 로직 진행...")
    log.info("===== SubA(REQUIRED) ROLLBACK =====")

    throw TxException()
}

// RequiresNewSubComponentB
@Transactional(propagation = Propagation.REQUIRES_NEW)
fun commit() {
    log.info("===== SubB(REQUIRES_NEW) BEGIN =====")
    log.info(">> SubB 로직 진행...")
    log.info("===== SubB(REQUIRES_NEW) COMMIT =====")
}

// Test
@Test
fun `SubA(Rollback) & SubB(Rollback)`() {
    shouldThrow<UnexpectedRollbackException> { main.case2() }
}
```

<div style="text-align: left">
  <img src="/assets/img/posts/2023-01-09-Transaction%20전파/requiresnew2.png" alt="img"/>
</div>

- SubA는 Unchecked Exception이 발생하였고 SubB는 정상적으로 Commit되었다
- 따라서 결과는 다음과 같이 진행된다
  - Main & SubA의 TX1 = `rollbackOnly 마킹`으로 인한 UnexpectedRollbackException + Rollback
  - SubB의 TX2 = Commit

<br>

### SUPPORTS

> 기존 트랜잭션 X: 트랜잭션 없이 진행<br>
> 기존 트랜잭션 O: 기존 트랜잭션에 참여
{: .prompt-info }

#### 1) 기존 트랜잭션 X

```kotlin
// SupportsMainComponent
fun case1() {
    log.info("===== Main(No TX) BEGIN =====")
    log.info(">> Main 로직 진행...")
    sub.execute()
    log.info("===== Main(No TX) COMMIT =====")
}

// SupportsSubComponent
@Transactional(propagation = Propagation.SUPPORTS)
fun execute() {
    log.info("===== Sub(SUPPORTS) BEGIN =====")
    log.info(">> Sub 로직 진행...")
    log.info("===== Sub(SUPPORTS) COMMIT =====")
}

// Test
@Test
fun `기존 트랜잭션 X`() {
    shouldNotThrowAny { main.case1() }
}
```

<div style="text-align: left">
  <img src="/assets/img/posts/2023-01-09-Transaction%20전파/supports1.png" alt="img"/>
</div>

- 기존 활성화 트랜잭션이 없으므로 `SUPPORTS`에서도 트랜잭션 없이 진행한다

#### 2) 기존 트랜잭션 O

```kotlin
// SupportsMainComponent
@Transactional
fun case2() {
    log.info("===== Main(No TX) BEGIN =====")
    log.info(">> Main 로직 진행...")
    sub.execute()
    log.info("===== Main(No TX) COMMIT =====")
}

// SupportsSubComponent
@Transactional(propagation = Propagation.SUPPORTS)
fun execute() {
    log.info("===== Sub(SUPPORTS) BEGIN =====")
    log.info(">> Sub 로직 진행...")
    log.info("===== Sub(SUPPORTS) COMMIT =====")
}

// Test
@Test
fun `기존 트랜잭션 O`() {
    shouldNotThrowAny { main.case2() }
}
```

<div style="text-align: left">
  <img src="/assets/img/posts/2023-01-09-Transaction%20전파/supports2.png" alt="img"/>
</div>

- 기존 활성화 트랜잭션이 존재하므로 `SUPPORTS`에서도 그대로 참여한다

<br>

### NOT_SUPPORTED

> 기존 트랜잭션 X: 트랜잭션 없이 진행<br>
> 기존 트랜잭션 O: 트랜잭션 없이 진행
{: .prompt-info }

#### 1) 기존 트랜잭션 X

```kotlin
// NotSupportedMainComponent
fun case1() {
    log.info("===== Main(No TX) BEGIN =====")
    log.info(">> Main 로직 진행...")
    sub.execute()
    log.info("===== Main(No TX) COMMIT =====")
}

// NotSupportedSubComponent
@Transactional(propagation = Propagation.NOT_SUPPORTED)
fun execute() {
    log.info("===== Sub(NOT_SUPPORTED) BEGIN =====")
    log.info(">> Sub 로직 진행...")
    log.info("===== Sub(NOT_SUPPORTED) COMMIT =====")
}

// Test
@Test
fun `기존 트랜잭션 X`() {
    shouldNotThrowAny { main.case1() }
}
```

<div style="text-align: left">
  <img src="/assets/img/posts/2023-01-09-Transaction%20전파/notsupported1.png" alt="img"/>
</div>

- 기존 활성화 트랜잭션이 없으므로 `NOT_SUPPORTED`에서도 트랜잭션 없이 진행한다

#### 2) 기존 트랜잭션 O

```kotlin
// NotSupportedMainComponent
@Transactional
fun case2() {
    log.info("===== Main(No TX) BEGIN =====")
    log.info(">> Main 로직 진행...")
    sub.execute()
    log.info("===== Main(No TX) COMMIT =====")
}

// NotSupportedSubComponent
@Transactional(propagation = Propagation.NOT_SUPPORTED)
fun execute() {
    log.info("===== Sub(NOT_SUPPORTED) BEGIN =====")
    log.info(">> Sub 로직 진행...")
    log.info("===== Sub(NOT_SUPPORTED) COMMIT =====")
}

// Test
@Test
fun `기존 트랜잭션 O`() {
    shouldNotThrowAny { main.case2() }
}
```

<div style="text-align: left">
  <img src="/assets/img/posts/2023-01-09-Transaction%20전파/notsupported2.png" alt="img"/>
</div>

- 기존 활성화 트랜잭션이 존재하더라도 `NOT_SUPPORTED`에서는 트랜잭션 없이 진행한다
- 기존 트랜잭션은 `잠시 중단 (suspending)`된다

<br>

### MANDATORY

> 기존 트랜잭션 X: `IllegalTransactionStateException` 예외 발생<br>
> 기존 트랜잭션 O: 기존 트랜잭션에 참여
{: .prompt-info }

- 반드시 기존 트랜잭션이 존재해야만 하고 없으면 예외가 발생한다

#### 1) 기존 트랜잭션 X

```kotlin
// MandatoryMainComponent
fun case1() {
    log.info("===== Main(No TX) BEGIN =====")
    log.info(">> Main 로직 진행...")
    sub.execute()
    log.info("===== Main(No TX) COMMIT =====")
}

// MandatorySubComponent
@Transactional(propagation = Propagation.MANDATORY)
fun execute() {
    log.info("===== Sub(MANDATORY) BEGIN =====")
    log.info(">> Sub 로직 진행...")
    log.info("===== Sub(MANDATORY) COMMIT =====")
}

// Test
@Test
fun `기존 트랜잭션 X`() {
    shouldThrow<IllegalTransactionStateException> { main.case1() }
}
```

<div style="text-align: left">
  <img src="/assets/img/posts/2023-01-09-Transaction%20전파/mandatory1.png" alt="img"/>
</div>

- `MANDATORY`는 기존 활성화 트랜잭션이 `반드시 존재`해야만 하고 위와 같이 존재하지 않으면 `IllegalTransactionStateException`이 발생한다

#### 2) 기존 트랜잭션 O

```kotlin
// MandatoryMainComponent
@Transactional
fun case2() {
    log.info("===== Main(No TX) BEGIN =====")
    log.info(">> Main 로직 진행...")
    sub.execute()
    log.info("===== Main(No TX) COMMIT =====")
}

// MandatorySubComponent
@Transactional(propagation = Propagation.MANDATORY)
fun execute() {
    log.info("===== Sub(MANDATORY) BEGIN =====")
    log.info(">> Sub 로직 진행...")
    log.info("===== Sub(MANDATORY) COMMIT =====")
}

// Test
@Test
fun `기존 트랜잭션 O`() {
    shouldNotThrowAny { main.case2() }
}
```

<div style="text-align: left">
  <img src="/assets/img/posts/2023-01-09-Transaction%20전파/mandatory2.png" alt="img"/>
</div>

- `MANDATORY`는 기존 활성화 트랜잭션이 존재하면 위와 같이 참여한다

<br>

### NEVER

> 기존 트랜잭션 X: 트랜잭션 없이 진행<br>
> 기존 트랜잭션 O: `IllegalTransactionStateException` 예외 발생
{: .prompt-info }

- MANDATORY와는 반대로 반드시 기존 트랜잭션이 없어야하고 있으면 예외가 발생한다

#### 1) 기존 트랜잭션 X

```kotlin
// NeverMainComponent
fun case1() {
    log.info("===== Main(No TX) BEGIN =====")
    log.info(">> Main 로직 진행...")
    sub.execute()
    log.info("===== Main(No TX) COMMIT =====")
}

// NeverSubComponent
@Transactional(propagation = Propagation.NEVER)
fun execute() {
    log.info("===== Sub(NEVER) BEGIN =====")
    log.info(">> Sub 로직 진행...")
    log.info("===== Sub(NEVER) COMMIT =====")
}

// Test
@Test
fun `기존 트랜잭션 X`() {
    shouldNotThrowAny { main.case1() }
}
```

<div style="text-align: left">
  <img src="/assets/img/posts/2023-01-09-Transaction%20전파/never1.png" alt="img"/>
</div>

- `NEVER`는 기존 활성화 트랜잭션이 `반드시 존재하지 않아야` 하고 위와 같이 존재하면 `IllegalTransactionStateException`이 발생한다

#### 2) 기존 트랜잭션 O

```kotlin
// NeverMainComponent
@Transactional
fun case2() {
    log.info("===== Main(No TX) BEGIN =====")
    log.info(">> Main 로직 진행...")
    sub.execute()
    log.info("===== Main(No TX) COMMIT =====")
}

// NeverSubComponent
@Transactional(propagation = Propagation.NEVER)
fun execute() {
    log.info("===== Sub(NEVER) BEGIN =====")
    log.info(">> Sub 로직 진행...")
    log.info("===== Sub(NEVER) COMMIT =====")
}

// Test
@Test
fun `기존 트랜잭션 O`() {
    shouldThrow<IllegalTransactionStateException> { main.case2() }
}
```

<div style="text-align: left">
  <img src="/assets/img/posts/2023-01-09-Transaction%20전파/never2.png" alt="img"/>
</div>

- `NEVER`는 기존 활성화 트랜잭션이 존재하지 않으면 위와 같이 트랜잭션이 없는 상태로 진행된다

<br>

### NESTED

> 기존 트랜잭션 X: 새로운 트랜잭션 생성<br>
> 기존 트랜잭션 O: `중첩 트랜잭션` 생성
{: .prompt-info }

- 중첩된 본인 트랜잭션은 외부 트랜잭션으로부터 영향을 받지만 영향을 주지는 않는다
  - 본인이 Rollback: 외부는 영향 X
  - 외부에서 Rollback: 본인도 Rollback

#### 1) 기존 트랜잭션 X

```kotlin
// NestedMainComponent
fun case1() {
    log.info("===== Main(No Tx) BEGIN =====")
    log.info(">> Main 로직 진행...")
    sub.commit()
    log.info("===== Main(No Tx) COMMIT =====")
}

// NestedSubComponent
@Transactional(propagation = Propagation.NESTED)
fun commit() {
    log.info("===== Sub(NESTED) BEGIN =====")
    log.info(">> Sub 로직 진행...")
    log.info("===== Sub(NESTED) COMMIT =====")
}

// Test
@Test
fun `기존 트랜잭션 X`() {
    shouldNotThrowAny { main.case1() }
}
```

<div style="text-align: left">
  <img src="/assets/img/posts/2023-01-09-Transaction%20전파/nested1.png" alt="img"/>
</div>

- `NESTED`는 기존 활성화 트랜잭션이 존재하지 않으면 `새로운 트랜잭션`을 생성해서 진행한다

#### 2) 기존 트랜잭션 O: Main(Commit) & Sub(Rollback)

```kotlin
// NestedMainComponent
@Transactional
fun case2(jpaTx: Boolean) {
    log.info("===== Main(REQUIRED) BEGIN =====")
    log.info(">> Main 로직 진행...")

    if (jpaTx) { // JPA
        sub.rollback()
    } else { // JDBC
        try {
            sub.rollback()
        } catch (_: Exception) {
        } finally {
            log.info("===== Main(REQUIRED) COMMIT =====")
        }
    }
}

// NestedSubComponent
@Transactional(propagation = Propagation.NESTED)
fun rollback() {
    log.info("===== Sub(NESTED) BEGIN =====")
    log.info(">> Sub 로직 진행...")
    log.info("===== Sub(NESTED) ROLLBACK =====")

    throw TxException()
}
```

##### JPA

```kotlin
@Test
fun `기존 트랜잭션 O - Main(Commit) & Sub(Rollback)`() {
    shouldThrow<NestedTransactionNotSupportedException> { main.case2(jpaTx = true) }
}
```

<div style="text-align: left">
  <img src="/assets/img/posts/2023-01-09-Transaction%20전파/nested2-jpa.png" alt="img"/>
</div>

- `JpaTransactionManager`는 `중첩 트랜잭션 NESTED`를 지원하지 않는다

##### JDBC

```kotlin
@Test
fun `기존 트랜잭션 O - Main(Commit) & Sub(Rollback)`() {
    shouldNotThrowAny { main.case2(jpaTx = false) }
}
```

<div style="text-align: left">
  <img src="/assets/img/posts/2023-01-09-Transaction%20전파/nested2-jdbc.png" alt="img"/>
</div>

- `JdbcTransactionManager`는 JpaTransactionManager와는 달리 `중첩 트랜잭션 NESTED`를 지원한다
- 결과로 보면 알 수 있듯이 `중첩 트랜잭션`은 외부 트랜잭션에 영향을 주지 않는다

#### 3) 기존 트랜잭션 O: Main(Rollback) & Sub(Commit)

```kotlin
// NestedMainComponent
@Transactional
fun case3() {
    log.info("===== Main(REQUIRED) BEGIN =====")
    log.info(">> Main 로직 진행...")
    sub.commit()
    log.info("===== Main(REQUIRED) ROLLBACK =====")

    throw TxException()
}

// NestedSubComponent
@Transactional(propagation = Propagation.NESTED)
fun commit() {
    log.info("===== Sub(NESTED) BEGIN =====")
    log.info(">> Sub 로직 진행...")
    log.info("===== Sub(NESTED) COMMIT =====")
}
```

##### JPA

```kotlin
@Test
fun `기존 트랜잭션 O - Main(Rollback) & Sub(Commit)`() {
    shouldThrow<NestedTransactionNotSupportedException> { main.case3() }
}
```

<div style="text-align: left">
  <img src="/assets/img/posts/2023-01-09-Transaction%20전파/nested3-jpa.png" alt="img"/>
</div>

- `JpaTransactionManager`는 `중첩 트랜잭션 NESTED`를 지원하지 않는다

##### JDBC

```kotlin
@Test
fun `기존 트랜잭션 O - Main(Rollback) & Sub(Commit)`() {
    shouldThrow<TxException> { main.case3() }
}
```

<div style="text-align: left">
  <img src="/assets/img/posts/2023-01-09-Transaction%20전파/nested2-jdbc.png" alt="img"/>
</div>

- `JdbcTransactionManager`는 JpaTransactionManager와는 달리 `중첩 트랜잭션 NESTED`를 지원한다
- 최종 결과는 외부 트랜잭션의 Unchecked Exception으로 인해 Rollback된다

<br>
> 관련된 코드는 [깃허브](https://github.com/sjiwon/devlog-codes/tree/main/spring-transaction/transaction-propagation)에서 확인할 수 있습니다
