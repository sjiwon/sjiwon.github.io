---
title: AspectJ 모듈 + Proxy 기반 애노테이션을 혼용했을 때 발생하는 문제
date: 2024-01-16 10:00 +0900
aliases: null
tags:
  - Spring
  - AOP
  - AOP 트랜잭션
  - Aspect Transactional
  - Redis Transaction
  - setEnableTransactionSupport
  - MULTI EXEC DISCARD
image: /assets/img/thumbnails/Spring.png
categories:
  - Skill
  - Spring
---

## 시나리오

요구사항에 대한 로직을 작성하다가 아래와 같은 생각이 들었다고 하자

- 너무나도 많은 흩어진 부분에서 공통적으로 적용되는 로직 존재
- 이러한 로직을 AOP(Aspect Oriented Programming)를 활용해서 공통 모듈화

위의 로직은 All or Nothing을 지켜야 하기 때문에 Transaction 처리가 필요하다고 가정하자

<br>
그러면 여기서 가장 심플하게 생각할 수 있는 구현 방안은 아래와 같다

1. Spring에서 제공해주는 `@Aspect`를 활용해서 Advice를 정의
2. 적절한 위치에 대한 `Pointcut`을 정의해서 AOP 적용
3. All or Nothing을 지키기 위해서 `Spring에서 제공해주는 @Transactional` 활용

```kotlin
@Aspect
@Component
class ExtractCommonLogicRdbTxAop(
    ...
) {
    @Transactional
    @Around("@annotation(...)")
    fun handle(joinPoint: ProceedingJoinPoint): Any {
        ...
        throw RuntimeException() // Unchecked Exception이니 위의 모든 로직은 rollback?
    }
}
```

> 과연 의도한대로 동작할까?
{: .prompt-danger }

<br>

## With RDB

### 1. @Around + @Transactional

Rdb를 활용하는 로직에 대해서 위에서 설명한 프로세스를 만들어보자

<span style="color:green">Member 도메인</span>

```kotlin
@Entity
@Table(name = "member")
class Member(
    @Id
    @GeneratedValue(strategy = IDENTITY)
    val id: Long = 0L,

    var name: String,
) {
    fun update(name: String) {
        this.name = name
    }
}

interface MemberRepository : JpaRepository<Member, Long>
```

<span style="color:green">AOP 로직</span>

```kotlin
@Aspect
@Component
class ExtractCommonLogicRdbTxAop(
    private val memberRepository: MemberRepository,
) {
    @Transactional
    @Around("@annotation(com.sjiwon.aspect.rdb.ExtractCommonLogicRdbTxTypeA)")
    fun typeA(joinPoint: ProceedingJoinPoint): Any {
        println("AOP - ExtractCommonLogicRdbTxTypeA")
        memberRepository.saveAll(
            listOf(
                Member(name = "MemberA"),
                Member(name = "MemberB"),
                Member(name = "MemberC"),
            )
        )
        throw RuntimeException()
    }
}
```

<span style="color:green">API Call</span>

```kotlin
@RestController
class RdbApi(
    private val rdbService: RdbService,
) {
    @PostMapping("/rdb/typeA")
    fun typeA(): String {
        rdbService.typeA()
        return "ok"
    }
}

@Service
class RdbService {
    @ExtractCommonLogicRdbTxTypeA
    fun typeA() {
    }
}
```

API Call 결과를 예측해보자

- `@Transactional`이 걸렸고 내부에서 `Unchecked Exception`이 발생했으니까 3건의 Insert는 모두 Rollback?


<div style="text-align: left">
  <img src="/assets/img/posts/2024-01-16-AspectJ%20모듈%20+%20Proxy%20기반%20애노테이션을%20혼용했을%20때%20발생하는%20문제/img1.png" alt="img"/>
</div>
<div style="text-align: left">
  <img src="/assets/img/posts/2024-01-16-AspectJ%20모듈%20+%20Proxy%20기반%20애노테이션을%20혼용했을%20때%20발생하는%20문제/img2.png" alt="img"/>
</div>

Unchecked Exception이 발생하였고 @Transactional을 적용했음에도 불구하고 Rollback이 되지 않고 Commit되었다<br>
이러한 결과가 도출된 이유를 분석해보자

#### Spring 공식문서 분석

[Spring AspectJ Docs](https://docs.spring.io/spring-framework/reference/core/aop/ataspectj/advice.html)

<div style="text-align: left">
  <img src="/assets/img/posts/2024-01-16-AspectJ%20모듈%20+%20Proxy%20기반%20애노테이션을%20혼용했을%20때%20발생하는%20문제/img3.png" alt="img"/>
</div>

- Spring AOP & AspectJ 둘 다 <span style="color:red">동일한 우선순위 규칙</span>을 따라 Advice 실행 순서를 결정 
  - 우선순위가 가장 높은 Advice가 먼저 실행
- <span style="color:red">서로 다른 @Aspect에서 정의한 Advice가 동일한 JoinPoint에서 실행</span>되어야 하는 경우 <ins>우선순위를 별도로 지정하지 않으면 실행 순서는 정의되지 않는다</ins> 
  - @Order 적용 or Ordered 인터페이스를 구현함으로써 실행 순서 정의

<br>
그렇다면 @Transactional의 우선순위는 무엇일까?

[Spring @Transactional Docs](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/annotations.html)

<div style="text-align: left">
  <img src="/assets/img/posts/2024-01-16-AspectJ%20모듈%20+%20Proxy%20기반%20애노테이션을%20혼용했을%20때%20발생하는%20문제/img4.png" alt="img"/>
</div>
<div style="text-align: left">
  <img src="/assets/img/posts/2024-01-16-AspectJ%20모듈%20+%20Proxy%20기반%20애노테이션을%20혼용했을%20때%20발생하는%20문제/img5.png" alt="img"/>
</div>

- @Transactional의 우선순위는 `Ordered.LOWEST_PRECEDENCE = 가장 후순위`이다

<br>

현재 상황을 요약해보자

- ExtractCommonLogicRdbAop, @Transactional은 모두 동일한 JoinPoint에서 실행
- ExtractCommonLogicRdbAop에는 어떠한 우선순위도 지정 X
- @Transactional은 가장 후순위

> 그렇다면 위의 문서에 의해서 ExtractCommonLogicRdbAop에 우선순위를 지정해서 실행 순서를 제어해보자

```kotlin
@Aspect
@Component
@Order(1) // 우선순위 제어?
class ExtractCommonLogicRdbTxAop(
    private val memberRepository: MemberRepository,
) {
    @Transactional
    @Around("@annotation(com.sjiwon.aspect.rdb.ExtractCommonLogicRdbTxTypeA)")
    fun typeA(joinPoint: ProceedingJoinPoint): Any {
        println("AOP - ExtractCommonLogicRdbTxTypeA")
        memberRepository.saveAll(
            listOf(
                Member(name = "MemberA"),
                Member(name = "MemberB"),
                Member(name = "MemberC"),
            )
        )
        throw RuntimeException()
    }
}
```

<div style="text-align: left">
  <img src="/assets/img/posts/2024-01-16-AspectJ%20모듈%20+%20Proxy%20기반%20애노테이션을%20혼용했을%20때%20발생하는%20문제/img6.png" alt="img"/>
</div>

- 그러나 여전히 원하는대로 동작은 되지 않고 있다

<br>
이 부분을 해결하기 위해서 위의 공식문서 마지막에 존재하는 Note를 읽어보자

<div style="text-align: left">
  <img src="/assets/img/posts/2024-01-16-AspectJ%20모듈%20+%20Proxy%20기반%20애노테이션을%20혼용했을%20때%20발생하는%20문제/img7.png" alt="img"/>
</div>

- 현재 @Around와 @Transactional은 동일한 @Aspect 내부에서 정의되고 동작하도록 구현하였다
- 그런데 문서에 나와있듯이 <span style="color:red">동일한 @Aspect 내부에서 정의한 Advice들은 리플렉션을 통해서 소스 코드 선언 순서를 검색할 방법이 없다</span>

결론적으로 위의 문제는 동일한 @Aspect 내부에서 서로 다른 Advice간의 순서를 지정하려고 했고 이는 Spring AOP 메커니즘 자체적으로 불가능한 로직이다<br>
따라서 문서에서도 권장하듯이 @Transactional을 적용하기 위한 로직을 분리해야 한다

### 2. @Around + SeparateRdbTransactionalComponent

```kotlin
@Aspect
@Component
class ExtractCommonLogicRdbTxAop(
    private val memberRepository: MemberRepository,
    private val separateRdbTransactionalComponent: SeparateRdbTransactionalComponent,
) {
    @Around("@annotation(com.sjiwon.aspect.rdb.ExtractCommonLogicRdbTxTypeB)")
    fun typeB(joinPoint: ProceedingJoinPoint): Any {
        println("AOP - ExtractCommonLogicRdbTxTypeB")
        separateRdbTransactionalComponent.invoke() // 트랜잭션 분리
        return joinPoint.proceed()
    }
}

@Component
class SeparateRdbTransactionalComponent(
    private val memberRepository: MemberRepository,
) {
    @Transactional
    fun invoke() {
        memberRepository.saveAll(
            listOf(
                Member(name = "MemberA"),
                Member(name = "MemberB"),
                Member(name = "MemberC"),
            )
        )
        throw RuntimeException()
    }
}
```

```kotlin
@RestController
class RdbApi(
    private val rdbService: RdbService,
) {
    @PostMapping("/rdb/typeB")
    fun typeB(): String {
        rdbService.typeB()
        return "ok"
    }
}

@Service
class RdbService {
    @ExtractCommonLogicRdbTxTypeB
    fun typeB() {
    }
}
```

<div style="text-align: left">
  <img src="/assets/img/posts/2024-01-16-AspectJ%20모듈%20+%20Proxy%20기반%20애노테이션을%20혼용했을%20때%20발생하는%20문제/img8.png" alt="img"/>
</div>

- 이제서야 원하는대로 동작하는 것을 확인할 수 있다

<br>
물론 `TransactionTemplate`을 활용해서 프로그래밍적으로 Tx Scope를 AspectJ 모듈 내부에서 적용해도 된다

```kotlin
@Aspect
@Component
class ExtractCommonLogicRdbTxAop(
    private val memberRepository: MemberRepository,
    private val separateRdbTransactionalComponent: SeparateRdbTransactionalComponent,
    private val transactionTemplate: TransactionTemplate,
) {
    @Around("@annotation(com.sjiwon.aspect.rdb.ExtractCommonLogicRdbTxTypeC)")
    fun typeC(joinPoint: ProceedingJoinPoint): Any {
        println("AOP - ExtractCommonLogicRdbTxTypeC")
        transactionTemplate.executeWithoutResult {
            memberRepository.saveAll(
                listOf(
                    Member(name = "MemberA"),
                    Member(name = "MemberB"),
                    Member(name = "MemberC"),
                )
            )
            throw RuntimeException()
        }
        return joinPoint.proceed()
    }
}
```

```kotlin
@RestController
class RdbApi(
    private val rdbService: RdbService,
) {
    @PostMapping("/rdb/typeC")
    fun typeC(): String {
        rdbService.typeC()
        return "ok"
    }
}

@Service
class RdbService {
    @ExtractCommonLogicRdbTxTypeC
    fun typeC() {
    }
}
```

<div style="text-align: left">
  <img src="/assets/img/posts/2024-01-16-AspectJ%20모듈%20+%20Proxy%20기반%20애노테이션을%20혼용했을%20때%20발생하는%20문제/img9.png" alt="img"/>
</div>

<br>

## With Redis

위의 결과로 알 수 있는 사실은 사용자의 커스텀한 AOP 로직에 @Transactional과 같은 Proxy 기반 메커니즘을 함께 적용하면 원하는대로 Tx Scope가 적용되지 않을 수 있고 컴포넌트 분리를 통해서 해결해야 한다<br>
이번에는 Redis의 Transaction 처리에 대해서 살펴보려고 한다

<br>
[Redis Transaction Docs](https://redis.io/docs/latest/develop/interact/transactions/)

Redis Transaction의 핵심 포인트는 아래와 같다

- Transaction으로 관리되는 모든 Command들은 직렬화되어 `순차적으로 실행`된다
- Transaction간의 Command들은 격리된 상태로 실행되도록 보장된다

<br>
Redis Transaction 진행 과정은 아래와 같다

1. `MULTI`를 통해서 Redis Transaction 시작
2. `EXEC/DISCARD`를 통해서 Redis Transaction Command들에 대한 작업 실행
   - EXEC = 모든 명령 실행
   - DISCARD = Transaction Queue가 Flush되고 종료

<br>
[Spring Redis Transaction](https://docs.spring.io/spring-data/redis/reference/redis/transactions.html)

Redis와 @Transactional을 같이 사용하려면 어떻게 해야할까?

- 기본적으로 RedisTemplate은 Spring Transaction에 참여할 수 없다
- 따라서 별도로 RedisTemplate을 빈으로 등록할 때 setEnableTransactionSupport를 설정해줘야 한다 (기본값 = false)

<br>
setEnableTransactionSupport를 설정하게 된다면 <span style="color:red">Redis Transaction의 MULTI -> ... -> EXEC/DISCARD 흐름</span>을 ThreadLocal 기반으로 내부적으로 관리한다

- ReadOnly Command
  - 현재 쓰레드에 바인딩되지 않은 `새로운 RedisConnection Pipeline`에서 진행된다
  - Transaction Queue에서 관리 X
- Writable Command
  - Transaction Queue의 관리를 받아서 제어된다

### Transaction 테스트

```kotlin
@Configuration
class RedisConfig(
    @Value("\${spring.data.redis.host}") val host: String,
    @Value("\${spring.data.redis.port}") val port: Int,
) {
    @Bean
    fun redisConnectionFactory(): RedisConnectionFactory {
        val redisStandaloneConfiguration = RedisStandaloneConfiguration(host, port)
        return LettuceConnectionFactory(redisStandaloneConfiguration)
    }

    @Bean
    fun redisTemplate(): RedisTemplate<String, Any> {
        return RedisTemplate<String, Any>().apply {
            connectionFactory = redisConnectionFactory()
            keySerializer = StringRedisSerializer()
        }
    }

    @Bean
    fun nonTxTemplate(): StringRedisTemplate {
        return StringRedisTemplate(redisConnectionFactory()).apply {
            keySerializer = StringRedisSerializer()
        }
    }

    @Bean
    fun txTemplate(): StringRedisTemplate {
        return StringRedisTemplate(redisConnectionFactory()).apply {
            keySerializer = StringRedisSerializer()
            setEnableTransactionSupport(true)
        }
    }
}
```

- `setEnableTransactionSupport` 설정 여부에 따라 TX 제어 차이를 알아보자

```kotlin
@Aspect
@Component
class ExtractCommonLogicRedisTxAop(
    @Qualifier("nonTxTemplate") private val nonTxTemplate: StringRedisTemplate,
    @Qualifier("txTemplate") private val txTemplate: StringRedisTemplate,
    private val separateNonTxTemplate: SeparateRedisTransactionalComponentA,
    private val separateTxTemplate: SeparateRedisTransactionalComponentB,
) {
    @Transactional
    @Around("@annotation(com.sjiwon.aspect.redis.ExtractCommonLogicRedisTxTypeA)")
    fun typeA(joinPoint: ProceedingJoinPoint): Any {
        println("AOP - ExtractCommonLogicRedisTxTypeA")
        val executor = nonTxTemplate.opsForValue()
        executor.set("typeA-nonTxTemplate-1", "success")
        executor.set("typeA-nonTxTemplate-2", "success")
        executor.set("typeA-nonTxTemplate-3", "success")
        throw RuntimeException()
    }

    @Transactional
    @Around("@annotation(com.sjiwon.aspect.redis.ExtractCommonLogicRedisTxTypeB)")
    fun typeB(joinPoint: ProceedingJoinPoint): Any {
        println("AOP - ExtractCommonLogicRedisTxTypeB")
        val executor = txTemplate.opsForValue()
        executor.set("typeB-nonTxTemplate-1", "success")
        executor.set("typeB-nonTxTemplate-2", "success")
        executor.set("typeB-nonTxTemplate-3", "success")
        throw RuntimeException()
    }

    @Around("@annotation(com.sjiwon.aspect.redis.ExtractCommonLogicRedisTxTypeC)")
    fun typeC(joinPoint: ProceedingJoinPoint): Any {
        println("AOP - ExtractCommonLogicRedisTxTypeC")
        separateNonTxTemplate.invoke()
        return joinPoint.proceed()
    }

    @Around("@annotation(com.sjiwon.aspect.redis.ExtractCommonLogicRedisTxTypeD)")
    fun typeD(joinPoint: ProceedingJoinPoint): Any {
        println("AOP - ExtractCommonLogicRedisTxTypeD")
        separateTxTemplate.invoke()
        return joinPoint.proceed()
    }
}

@Component
class SeparateRedisTransactionalComponentA(
    @Qualifier("nonTxTemplate") private val template: StringRedisTemplate,
) {
    @Transactional
    fun invoke() {
        val executor = template.opsForValue()
        executor.set("separate-component-nonTxTemplate-1", "success")
        executor.set("separate-component-nonTxTemplate-2", "success")
        executor.set("separate-component-nonTxTemplate-3", "success")
        throw RuntimeException()
    }
}

@Component
class SeparateRedisTransactionalComponentB(
    @Qualifier("txTemplate") private val template: StringRedisTemplate,
) {
    @Transactional
    fun invoke() {
        val executor = template.opsForValue()
        executor.set("separate-component-txTemplate-1", "success")
        executor.set("separate-component-txTemplate-2", "success")
        executor.set("separate-component-txTemplate-3", "success")
        throw RuntimeException()
    }
}
```

RDB 테스트 결과와 위의 Redis Transaction 설명을 토대로 결과를 예측해보자

- TypeA = Tx 관리 X
- TypeB = Tx 관리 X
- TypeC = Tx 관리 X
- TypeD = Tx 관리 O

<span style="color:green">TypeA</span>

<div style="text-align: left">
  <img src="/assets/img/posts/2024-01-16-AspectJ%20모듈%20+%20Proxy%20기반%20애노테이션을%20혼용했을%20때%20발생하는%20문제/img10.png" alt="img"/>
</div>
<div style="text-align: left">
  <img src="/assets/img/posts/2024-01-16-AspectJ%20모듈%20+%20Proxy%20기반%20애노테이션을%20혼용했을%20때%20발생하는%20문제/img11.png" alt="img"/>
</div>

<span style="color:green">TypeB</span>

<div style="text-align: left">
  <img src="/assets/img/posts/2024-01-16-AspectJ%20모듈%20+%20Proxy%20기반%20애노테이션을%20혼용했을%20때%20발생하는%20문제/img12.png" alt="img"/>
</div>
<div style="text-align: left">
  <img src="/assets/img/posts/2024-01-16-AspectJ%20모듈%20+%20Proxy%20기반%20애노테이션을%20혼용했을%20때%20발생하는%20문제/img13.png" alt="img"/>
</div>

<span style="color:green">TypeC</span>

<div style="text-align: left">
  <img src="/assets/img/posts/2024-01-16-AspectJ%20모듈%20+%20Proxy%20기반%20애노테이션을%20혼용했을%20때%20발생하는%20문제/img14.png" alt="img"/>
</div>
<div style="text-align: left">
  <img src="/assets/img/posts/2024-01-16-AspectJ%20모듈%20+%20Proxy%20기반%20애노테이션을%20혼용했을%20때%20발생하는%20문제/img15.png" alt="img"/>
</div>

- Spring Transaction은 Rollback되었다
- 하지만 `setEnableTransactionSupport = false`로 설정된 Template을 활용해서 명령을 보냈고 Redis Transaction은 기본적으로 Spring Transaction에 참여하지 않기 때문에 Redis Command들간의 Transaction은 적용되지 않았다

<span style="color:green">TypeD</span>

<div style="text-align: left">
  <img src="/assets/img/posts/2024-01-16-AspectJ%20모듈%20+%20Proxy%20기반%20애노테이션을%20혼용했을%20때%20발생하는%20문제/img16.png" alt="img"/>
</div>
<div style="text-align: left">
  <img src="/assets/img/posts/2024-01-16-AspectJ%20모듈%20+%20Proxy%20기반%20애노테이션을%20혼용했을%20때%20발생하는%20문제/img17.png" alt="img"/>
</div>

- 정상적으로 Write Command들이 Redis Transaction Queue에 의해 관리되고 Redis Command들의 Rollback이 이루어짐을 확인할 수 있다
  - `setEnableTransactionSupport = true`로 설정함에 따라 Spring Transaction에 의해 관리된다
  - Transaction 내부에서 발생한 예외로 인해 `DISCARD` 명령어가 실행되기 때문에 Transaction Queue의 명령어들이 실행되지 않는것이다

<br>
> 관련된 코드는 [깃허브](https://github.com/sjiwon/devlog-codes/tree/main/spring/aop/aspect-module-with-proxy-based-annotation)에서 확인할 수 있습니다
