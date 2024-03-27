---
title: Transaction 추상화
date: 2023-01-09 10:00 +0900
aliases: null
tags: [ Spring, Transaction, 트랜잭션 추상화, PlatformTransactionManager, UnexpectedRollbackException ]
image: /assets/img/thumbnails/Spring.png
categories: [ Skill, Spring ]
---

## 트랜잭션?

> 더이상 쪼갤 수 없는 논리적 최소 작업 단위

- 논리적 작업 단위의 `All or Nothing` 보장

```kotlin
// 사용자 가입 로직
fun logic() {
    memberRepository.save(...) // 사용자 정보 저장
    bucketRepository.save(...) // 사용자 전용 버킷 저장
    ...
}
```

사용자 가입을 진행하기 위한 위의 로직은 `하나의 트랜잭션`으로 묶여있고 따라서 내부 로직들은 `All or Nothing`을 보장해야 한다<br>
그런데 중간에 어떠한 이유로 인해 특정 로직이 실패하게 된다면 트랜잭션 단위의 모든 로직은 Rollback되어야 한다

### 순수 JDBC vs ORM(JPA) 트랜잭션 처리 방식

자바를 활용해서 웹 애플리케이션을 개발할 때 DB에 접근하기 위해서는 JDBC라는 API를 활용해야 한다<br>
그리고 위에서 말한 Transaction에 대한 Commit/Rollback 처리 역시 JDBC Connection을 통해서 처리한다

> 밑에 나오는 예시들은 단순한 트랜잭션 제어 방식의 차이만 알아보는 코드이므로 Resource에 대한 clear 코드는 생략하였다
{: .prompt-warning }

#### 순수 JDBC를 활용한 제어

순수 JDBC (DataSource, Connection ,...) 등을 활용해서 관련된 로직을 작성하고 트랜잭션을 제어해보자

```kotlin
@Component
class JdbcComponent(
    private val dataSource: DataSource,
) {
    fun logic(exception: Boolean) {
        var connection: Connection? = null
        var pstmt: PreparedStatement?

        try {
            connection = dataSource.connection
            connection.autoCommit = false

            // 1. 사용자 정보 저장
            pstmt = connection.prepareStatement(
                "INSERT INTO members(name) VALUES (?)",
                PreparedStatement.RETURN_GENERATED_KEYS,
            )
            pstmt.setString(1, "Member")
            pstmt.executeUpdate()

            // 2. 사용자 개인 사물함 정보 저장
            val rs: ResultSet = pstmt.generatedKeys
            if (rs.next()) {
                pstmt = connection.prepareStatement(
                    "INSERT INTO buckets(member_id, capacity) VALUES (?, ?)",
                    PreparedStatement.RETURN_GENERATED_KEYS,
                )
                pstmt.setLong(1, rs.getLong(1))
                pstmt.setInt(2, 10)
                pstmt.executeUpdate()
            }

            // 예외 발생? All or Nothing?
            if (exception) {
                throw RuntimeException()
            }

            connection.commit()
        } catch (ex: Exception) {
            connection?.rollback()
        }
    }
}
```

#### ORM (JPA Hibernate)를 활용한 제어

자바 진영의 대표적인 ORM인 JPA(Hibernate)를 활용해서 관련된 로직을 작성하고 트랜잭션을 제어해보자

```kotlin
@Component
class JpaComponent(
    private val emf: EntityManagerFactory,
) {
    fun logic(exception: Boolean) {
        val em: EntityManager = emf.createEntityManager()
        val tx: EntityTransaction = em.transaction

        try {
            tx.begin()

            // 1. 사용자 정보 저장
            val member = Member(name = "Member")
            em.persist(member)

            // 2. 사용자 개인 사물함 정보 저장
            val bucket = Bucket(memberId = member.id, capacity = 10)
            em.persist(bucket)

            // 예외 발생? All or Nothing?
            if (exception) {
                throw RuntimeException()
            }

            tx.commit()
        } catch (ex: Exception) {
            tx.rollback()
        }
    }
}
```

#### 트랜잭션 제어 테스트

```kotlin
@SpringBootTest
@ExtendWith(DatabaseCleanerEachCallbackExtension::class)
@TestConstructor(autowireMode = TestConstructor.AutowireMode.ALL)
class TransactionHandlingTest(
    private val jdbcComponent: JdbcComponent,
    private val jpaComponent: JpaComponent,
    private val memberRepository: MemberRepository,
    private val bucketRepository: BucketRepository,
) {
    @Test
    fun `JdbcComponent - 예외 발생 X`() {
        jdbcComponent.logic(exception = false)

        assertSoftly {
            memberRepository.findAll() shouldHaveSize 1
            bucketRepository.findAll() shouldHaveSize 1
        }
    }

    @Test
    fun `JdbcComponent - 예외 발생 O`() {
        jdbcComponent.logic(exception = true)

        assertSoftly {
            memberRepository.findAll() shouldHaveSize 0
            bucketRepository.findAll() shouldHaveSize 0
        }
    }

    @Test
    fun `JpaComponent - 예외 발생 X`() {
        jpaComponent.logic(exception = false)

        assertSoftly {
            memberRepository.findAll() shouldHaveSize 1
            bucketRepository.findAll() shouldHaveSize 1
        }
    }

    @Test
    fun `JpaComponent - 예외 발생 O`() {
        jpaComponent.logic(exception = true)

        assertSoftly {
            memberRepository.findAll() shouldHaveSize 0
            bucketRepository.findAll() shouldHaveSize 0
        }
    }
}
```

<div style="text-align: left">
  <img src="/assets/img/posts/2023-01-09-Transaction%20추상화/img1.png" alt="img"/>
</div>

JDBC API & JPA 각각 트랜잭션을 다루는 방식에는 눈에 보이는 차이가 존재한다

- `JDBC API`: Connection 획득하고 autoCommit 설정하고 ...
- `JPA`: EntityManagerFactory로부터 EntityManager 생성하고 EntityManager로부터 EntityTransaction 생성하고 ...

기존 팀에서 순수 JDBC API를 활용해서 개발하다가 개발 생산성이 눈에 띄게 저하됨을 팀원 전부가 인지함에 따라 JPA라는 ORM 기술을 도입하는 결정을 내렸다고 하자<br>
많은 부분이 변경되겠지만 위의 예시와 같이 트랜잭션 관리에 대한 메커니즘 자체도 전부 변하게 될 것이다

Spring에서는 이러한 문제를 어떻게 해결하고 있을까?

## PlatformTransactionManager

<div style="text-align: left">
  <img src="/assets/img/posts/2023-01-09-Transaction%20추상화/img2.png" alt="img"/>
</div>

> 여러 DB 접근 기술의 `트랜잭션 매커니즘`을 추상화시킨 컴포넌트

- Spring Boot에서는 ***의존성, 라이브러리, yml 설정, ..*** 등을 확인해서 적절한 TransactionManager를 스프링 빈으로 등록한다

### getTransaction

<div style="text-align: left">
  <img src="/assets/img/posts/2023-01-09-Transaction%20추상화/img3.png" alt="img"/>
</div>

> 현재 활성화된 트랜잭션 획득 or 새로운 트랜잭션 생성
{: .prompt-info }

- Transaction Propagation의 기본값은 `REQUIRED`이다
  - 기존 트랜잭션 O: 해당 트랜잭션에 참여
  - 기존 트랜잭션 X: 새로운 트랜잭션 생성
- 따라서 `getTransaction`의 동작은 Transaction Propagation에 의해 결정된다

<br>
여기서 추가적으로 알아야 할 점은 `TransactionDefinition`이다<br>

- TransactionDefinition은 `반드시 새로운 트랜잭션`에 적용해야 원하는대로 동작한다
- 기존에 활성화된 트랜잭션에 대해서 TransactionDefinition을 넘겨준다고 하더라도 해당 설정으로 기존 트랜잭션의 속성이 변경되지는 않는다

```kotlin
@Component
class TransactionDef(
    private val transactionManager: PlatformTransactionManager,
) {
    fun execute() {
        val writebleTx = TransactionTemplate(transactionManager).apply {
            isReadOnly = false
        }
        val readOnlyTx = TransactionTemplate(transactionManager).apply {
            isReadOnly = true
        }

        println("## WritableTx -> ReadOnlyTx ##")
        writebleTx.executeWithoutResult {
            println("1. ReadOnly = ${TransactionSynchronizationManager.isCurrentTransactionReadOnly()}")
            readOnlyTx.executeWithoutResult {
                println("2. ReadOnly = ${TransactionSynchronizationManager.isCurrentTransactionReadOnly()}")
            }
        }

        println("\n## ReadOnlyTx -> WritableTx ##")
        readOnlyTx.executeWithoutResult {
            println("1. ReadOnly = ${TransactionSynchronizationManager.isCurrentTransactionReadOnly()}")
            writebleTx.executeWithoutResult {
                println("2. ReadOnly = ${TransactionSynchronizationManager.isCurrentTransactionReadOnly()}")
            }
        }
    }
}
```

<div style="text-align: left">
  <img src="/assets/img/posts/2023-01-09-Transaction%20추상화/img4.png" alt="img"/>
</div>

- writableTx & readOnlyTx 모두 `Propagation = REQUIRED`이므로 열린 트랜잭션에 참여한다
- 결과로 알 수 있듯이 각각의 Tx는 `readOnly` 속성이 다른데 Nested로 호출된다고 하더라도 이미 열린 트랜잭션의 속성을 따르게 된다

### commit

<div style="text-align: left">
  <img src="/assets/img/posts/2023-01-09-Transaction%20추상화/img5.png" alt="img"/>
</div>

> 트랜잭션 `커밋`
{: .prompt-info }

commit 시점에 가장 중요한 값은 `TransactionStatus's rollbackOnly`이다

- rollbackOnly가 true면 최종 물리 트랜잭션에서 아무리 Commit을 하려고 시도해도 `UnexpectedRollbackException`과 함께 전체 트랜잭션이 롤백된다

### rollback

<div style="text-align: left">
  <img src="/assets/img/posts/2023-01-09-Transaction%20추상화/img6.png" alt="img"/>
</div>

> 트랜잭션 `롤백`
{: .prompt-info }

`TransactionStatus's newTransaction` 값에 따라 내부적으로 다른 동작이 진행된다

#### newTransaction = true

newTransaction=true는 해당 트랜잭션이 `신규(새로 생성된 - 시작 지점의) 트랜잭션`이라는 의미이다<br>
시작 지점의 트랜잭션은 `물리적 트랜잭션`으로써 직접적으로 트랜잭션 동작에 관여할 수 있다

- 물리 트랜잭션 rollback: 내부 논리 트랜잭션 전체가 commit이더라도 최종 rollback
- 물리 트랜잭션 commit: 내부 논리 트랜잭션 commit/rollback 여부에 따라 최종 commit/rollback
  - 내부 논리 트랜잭션의 rollbackOnly 마킹으로 인해 물리 트랜잭션이 commit을 하려고 시도해도 `UnexpectedRollbackException`과 함께 최종 rollback된다

#### newTransaction = false

newTransaction=false는 해당 트랜잭션이 `기존 트랜잭션에 참여한 트랜잭션`이라는 의미이다<br>
해당 트랜잭션은 물리 트랜잭션이 아니라 논리적 트랜잭션이므로 직접적으로 트랜잭션 동작에 관여할 수는 없지만 `rollbackOnly Marking`을 통해서 간접적으로 rollback을 유도할 수 있다
