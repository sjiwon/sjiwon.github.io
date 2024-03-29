---
title: synchronized 키워드를 활용한 동시성 문제 해결 & 한계
date: 2023-04-04 13:00 +0900
aliases: null
tags:
  - Spring
  - synchronized 동시성
  - synchronized @Transactional 문제점
  - 분산 환경 synchronized
  - Spring Transactional Proxy
  - Nginx Reverse Proxy
  - Pessimistic Lock
image: /assets/img/thumbnails/Spring.png
categories:
  - Skill
  - Spring
---

## 동시성 처리

웹 서비스를 개발하다보면 수많은 종류의 동시성 문제를 경험해볼 수 있다
- 주문 도메인 상품 재고 동시성 처리
- 선착순 쿠폰 동시성 처리
- ...

동시성 문제는 결국 `멀티 쓰레드 환경`에서 공유 자원이 존재하는 Critical Section에 대한 Race Condition으로 인해 발생하는 문제이다<br>
본 포스팅에서는 `synchronized`라는 JVM에서 제공해주는 키워드를 통해서 동시성을 제어해보고 한계에 대해서 설명할 예정이다

### Process? Thread?

synchronized 키워드에 대해 알아보기전에 먼저 Process와 Thread의 개념부터 알아보자

<div style="text-align: left">
  <img src="/assets/img/posts/2023-04-04-synchronized%20키워드를%20활용한%20동시성%20문제%20해결%20&%20한계/img1.png" alt="img"/>
</div>

idea64.exe는 디스크에 저장된 실행 가능한 코드와 관련 데이터 파일로 구성된 `프로그램`이다<br>
이 idea64.exe를 더블클릭하게 되면

<div style="text-align: left">
  <img src="/assets/img/posts/2023-04-04-synchronized%20키워드를%20활용한%20동시성%20문제%20해결%20&%20한계/img2.png" alt="img"/>
</div>

디스크에 존재하던 프로그램이 `주기억장치에 적재`됨에 따라 프로세스가 되는 것이다
- 적재된 프로세스는 `Ready Queue에서 대기(Ready Status)`하게 되고 CPU 스케줄링에 의해서 CPU를 할당받게 되면 Running Status가 된다

> 프로그램 &rarr; 파일 시스템에 설치되어 있는 파일<br>
> 프로세스 &rarr; 메모리에 적재된 프로그램 & OS로부터 자원을 할당받는 `작업의 단위`<br>
> 쓰레드 &rarr; 프로세스가 할당받은 자원을 이용하는 `실행 흐름 단위`
{: .prompt-info }

하나의 프로세스에는 `최소 1개 이상의 쓰레드`가 존재하고 쓰레드가 실제 작업 실행의 주체이다<br>
프로세스 내부의 쓰레드들은 `프로세스의 Code, Datd, Heap을 공유`하고 `Stack, Register를 별도로 할당`받는다

<br>
각각의 쓰레드들이 `공유 자원에 동시에 접근`하게 되면 해당 문제를 `Race Condition`이라고 하고 Race Condition에 대한 동기화 메커니즘으로는 `Mutex, Semaphore, Monitor, ..`등이 존재한다

### synchronized 키워드

자바의 synchronized 키워드는 N개의 Thread가 동시에 공유 자원에 접근하는 것을 제어해서 Race Condition을 방지하는 동기화 메커니즘을 제공한다

- Critical Section에 대한 `Monitor Locking 메커니즘`을 통해서 제어한다

> 자바의 모든 객체는 `모니터 락(Monitor Lock)`을 가지고 있으며, 이를 통해 쓰레드 동기화를 수행할 수 있다.<br>
> synchronized 키워드는 객체의 모니터 락을 사용하여 `상호 배제(Mutual Exclusion)를 보장`한다.<br>
> 따라서 한 번에 하나의 쓰레드만이 Critical Section에 접근할 수 있다.
{: .prompt-tip }

<br>

## 동시성 제어 실습

```kotlin
@Entity
@Table(name = "ticket")
class Ticket(
    @Id
    @GeneratedValue(strategy = IDENTITY)
    val id: Long = 0L,

    var stock: Int,
) {
    fun purchase(amount: Int) {
        if (stock == 0) {
            throw RuntimeException("티켓이 매진되었습니다.")
        }
        if (stock < amount) {
            throw RuntimeException("티켓 재고가 부족합니다.")
        }
        stock -= amount
    }
}

interface TicketRepository : JpaRepository<Ticket, Long>
```

### 1. synchronized 적용 X

```kotlin
@Service
class TicketV1Service(
    private val ticketRepository: TicketRepository,
) {
    private val log: Logger = logger()

    @Transactional
    fun purchase(
        ticketId: Long,
        amount: Int,
    ) {
        val ticket = ticketRepository.findByIdOrNull(ticketId)
            ?: throw RuntimeException("Ticket not found ... $ticketId")
        log.info("${Thread.currentThread().name} -> [Ticket${ticketId} 현재 보유량=${ticket.stock} & 구매 요청량=${amount}]")
        ticket.purchase(amount)
    }
}
```

<div style="text-align: left">
  <img src="/assets/img/posts/2023-04-04-synchronized%20키워드를%20활용한%20동시성%20문제%20해결%20&%20한계/img3.png" alt="img"/>
</div>
<div style="text-align: left">
  <img src="/assets/img/posts/2023-04-04-synchronized%20키워드를%20활용한%20동시성%20문제%20해결%20&%20한계/img4.png" alt="img"/>
</div>

- 현재 티켓 구매 로직에는 어떠한 동시성 처리도 적용되지 않았다
- 따라서 결과로 알 수 있듯이 `200명의 사용자 X 5장의 티켓 = 100장의 티켓`이 팔려야 정상인데 15장밖에 팔리지 않은것으로 기록되었다

### 2. synchronized 적용 O

```kotlin
@Service
class TicketV2Service(
    private val ticketRepository: TicketRepository,
) {
    private val log: Logger = logger()

    @Transactional
    @Synchronized
    fun purchase(
        ticketId: Long,
        amount: Int,
    ) {
        val ticket = ticketRepository.findByIdOrNull(ticketId)
            ?: throw RuntimeException("Ticket not found ... $ticketId")
        log.info("${Thread.currentThread().name} -> [Ticket${ticketId} 현재 보유량=${ticket.stock} & 구매 요청량=${amount}]")
        ticket.purchase(amount)
    }
}
```

<div style="text-align: left">
  <img src="/assets/img/posts/2023-04-04-synchronized%20키워드를%20활용한%20동시성%20문제%20해결%20&%20한계/img5.png" alt="img"/>
</div>
<div style="text-align: left">
  <img src="/assets/img/posts/2023-04-04-synchronized%20키워드를%20활용한%20동시성%20문제%20해결%20&%20한계/img6.png" alt="img"/>
</div>

- synchronized 키워드를 적용했음에도 불구하고 여전히 동시성 처리가 되지 않음을 확인할 수 있다
- 이유는 바로 `@Transactional의 동작원리` 때문이다

#### @Transactional 동작원리

<div style="text-align: left">
  <img src="/assets/img/posts/2023-04-04-synchronized%20키워드를%20활용한%20동시성%20문제%20해결%20&%20한계/img7.png" alt="img"/>
</div>

- @Transaction이 적용된 클래스는 `CGLIB`에 의해서 런타임에 `해당 클래스 기반 프록시`가 생성된다
- 그리고 @Transactional 로직으로 진입하기 전/후에서 Transaction Begin & Commit/Rollback이 진행되는 것이다

<div style="text-align: left">
  <img src="/assets/img/posts/2023-04-04-synchronized%20키워드를%20활용한%20동시성%20문제%20해결%20&%20한계/img8.png" alt="img"/>
</div>

- 이렇게 @Transactional이 걸려있는 비즈니스 로직에 synchronized 키워드를 붙이게 된다면 다음과 같이 동작한다

<div style="text-align: left">
  <img src="/assets/img/posts/2023-04-04-synchronized%20키워드를%20활용한%20동시성%20문제%20해결%20&%20한계/img9.png" alt="img"/>
</div>

- 해당 비즈니스 로직에 synchronized가 걸려있으니 해당 로직으로 진입할때 Monitor Lock을 가지고 진입하게 되는것이다

<div style="text-align: left">
  <img src="/assets/img/posts/2023-04-04-synchronized%20키워드를%20활용한%20동시성%20문제%20해결%20&%20한계/img10.png" alt="img"/>
</div>

- 그러면 Thread1을 제외한 나머지 쓰레드들은 비즈니스 로직에 접근하지 못하고 Lock을 얻기 위해서 대기한다
- 여기서 Thread1이 비즈니스 로직을 끝내고 커밋/롤백 시점으로 돌입한다고 가정하자

<div style="text-align: left">
  <img src="/assets/img/posts/2023-04-04-synchronized%20키워드를%20활용한%20동시성%20문제%20해결%20&%20한계/img11.png" alt="img"/>
</div>

- 이 시점에 Thread2가 진입하게 되면 아직 Thread1의 로직이 commit되기 전이므로 DB에 존재하는 Ticket의 stock은 여전히 100이다
- 그에 따라서 Thread2는 Ticket의 stock을 100으로 받게 되고 그에 따른 로직이 진행된다

이러한 이유로 인해서 `@Transactional + synchronized`는 Spring의 Proxy 메커니즘으로 인해 동시성 문제를 해결하지 못하는 것이다

### 3. synchronized & @Transactional 분리

#### 1) Facade Layer

```kotlin
@Component
class TicketFacade(
    private val target: TicketV3Service,
) {
    @Synchronized
    fun invoke(
        ticketId: Long,
        amount: Int,
    ) {
        target.purchase(ticketId, amount)
    }
}

@Service
class TicketV3Service(
    private val ticketRepository: TicketRepository,
) {
    private val log: Logger = logger()

    @Transactional
    fun purchase(
        ticketId: Long,
        amount: Int,
    ) {
        val ticket = ticketRepository.findByIdOrNull(ticketId)
            ?: throw RuntimeException("Ticket not found ... $ticketId")
        log.info("${Thread.currentThread().name} -> [Ticket${ticketId} 현재 보유량=${ticket.stock} & 구매 요청량=${amount}]")
        ticket.purchase(amount)
    }
}
```

<div style="text-align: left">
  <img src="/assets/img/posts/2023-04-04-synchronized%20키워드를%20활용한%20동시성%20문제%20해결%20&%20한계/img12.png" alt="img"/>
</div>
<div style="text-align: left">
  <img src="/assets/img/posts/2023-04-04-synchronized%20키워드를%20활용한%20동시성%20문제%20해결%20&%20한계/img13.png" alt="img"/>
</div>
<div style="text-align: left">
  <img src="/assets/img/posts/2023-04-04-synchronized%20키워드를%20활용한%20동시성%20문제%20해결%20&%20한계/img14.png" alt="img"/>
</div>

#### 2) 명시적 saveAndFlush

```kotlin
@Service
class TicketV4Service(
    private val ticketRepository: TicketRepository,
) {
    private val log: Logger = logger()

    @Synchronized
    fun purchase(
        ticketId: Long,
        amount: Int,
    ) {
        val ticket = ticketRepository.findByIdOrNull(ticketId)
            ?: throw RuntimeException("Ticket not found ... $ticketId")
        log.info("${Thread.currentThread().name} -> [Ticket${ticketId} 현재 보유량=${ticket.stock} & 구매 요청량=${amount}]")
        ticket.purchase(amount)
        ticketRepository.saveAndFlush(ticket)
    }
}
```

<div style="text-align: left">
  <img src="/assets/img/posts/2023-04-04-synchronized%20키워드를%20활용한%20동시성%20문제%20해결%20&%20한계/img15.png" alt="img"/>
</div>
<div style="text-align: left">
  <img src="/assets/img/posts/2023-04-04-synchronized%20키워드를%20활용한%20동시성%20문제%20해결%20&%20한계/img16.png" alt="img"/>
</div>

<br>

## 분산환경에서의 synchronized 한계점

synchronized 키워드는 `단일 프로세스` 상에서 특정 Critical Section에 대해서 멀티 쓰레드 동시성 문제를 해결할 수 있다<br>
하지만 SPOF, 트래픽 부하 분산, ..등 여러가지 이유로 WAS를 분산시켰다면 synchronized가 정상적으로 동작할까?

### 실습

간단하게 로컬환경에서 Docker를 활용해서 WAS 2대를 띄운 후 nginx를 통한 로드밸런싱을 적용해서 분산환경에서 synchronized만으로는 동시성 처리가 되지 않는지 확인하자

#### application.yml & Docker

```yaml
# application.yml
spring:
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://database:3306/ticket
    username: root
    password: 1234

  jpa:
    open-in-view: false
    hibernate:
      ddl-auto: create
    properties:
      hibernate:
        default_batch_fetch_size: 100

---
spring:
  config:
    activate:
      on-profile: server1

server: server1

---
spring:
  config:
    activate:
      on-profile: server2

server: server2
```

```dockerfile
# Server1
FROM amazoncorretto:17-alpine-jdk

WORKDIR /app

COPY /build/libs/ticket-concurrency-with-synchronized-0.0.1-SNAPSHOT.jar app.jar

ENV TZ=Asia/Seoul

ENTRYPOINT ["java", "-Dspring.profiles.active=server1", "-jar", "app.jar"]

# Server2
FROM amazoncorretto:17-alpine-jdk

WORKDIR /app

COPY /build/libs/ticket-concurrency-with-synchronized-0.0.1-SNAPSHOT.jar app.jar

ENV TZ=Asia/Seoul

ENTRYPOINT ["java", "-Dspring.profiles.active=server2", "-jar", "app.jar"]
```

```yaml
# docker-compose.yml
version: "3"
services:
  was1:
    build:
      context: .
      dockerfile: Dockerfile.server1
    container_name: was1
    restart: on-failure
    ports:
      - "8080:8080"
    networks:
      - application

  was2:
    build:
      context: .
      dockerfile: Dockerfile.server2
    container_name: was2
    restart: on-failure
    ports:
      - "8081:8080"
    networks:
      - application

  database:
    image: mysql:8.0.33
    container_name: database
    restart: always
    ports:
      - "13306:3306"
    environment:
      MYSQL_ROOT_PASSWORD: "1234"
      MYSQL_DATABASE: "ticket"
      TZ: "Asia/Seoul"
      LANG: "C.UTF_8"
    command:
      - --character-set-server=utf8mb4
      - --collation-server=utf8mb4_unicode_ci
      - --skip-character-set-client-handshake
    networks:
      - application

  nginx:
    image: nginx:1.21.5-alpine
    container_name: nginx
    ports:
      - "80:80"
    volumes:
      - ./app.conf:/etc/nginx/conf.d/default.conf
    depends_on:
      - was1
      - was2
    networks:
      - application

networks:
  application:
    external: true
```

```nginx
upstream backend {
    server was1:8080;
    server was2:8080;
}

server {
    listen 80;

    location / {
        proxy_pass http://backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

#### 로드밸런싱 테스트

<div style="text-align: left">
  <img src="/assets/img/posts/2023-04-04-synchronized%20키워드를%20활용한%20동시성%20문제%20해결%20&%20한계/loadbalancing.gif" alt="img"/>
</div>

#### API 로직

```kotlin
@RestController
class TicketApi(
    private val environment: Environment,
    private val logic: TicketV5Service,
) {
    private var visitCount = 0

    @GetMapping("/api/health")
    fun health(): Map<String, Any> = mapOf(
        "visitCount" to visitCount++,
        "server" to getServer()
    )

    data class Request(
        val amount: Int,
    )

    data class Response(
        val server: String,
        val ticket: Ticket,
    )

    @PostMapping("/api/v1/tickets/{ticketId}/purchase")
    fun purchase(
        @PathVariable ticketId: Long,
        @RequestBody request: Request,
    ): Response {
        val ticket = logic.purchase(ticketId, request.amount)
        return Response(
            server = getServer(),
            ticket = ticket,
        )
    }

    private fun getServer(): String = environment.getProperty("server", "?")

    @ExceptionHandler
    fun handle(ex: RuntimeException): String = ex.message ?: "empty"
}

@Service
class TicketV5Service(
    private val ticketRepository: TicketRepository,
) {
    private val log: Logger = logger()

    @Synchronized
    fun purchase(
        ticketId: Long,
        amount: Int,
    ): Ticket {
        val ticket = ticketRepository.findByIdOrNull(ticketId)
            ?: throw RuntimeException("Ticket not found ... $ticketId")
        log.info("${Thread.currentThread().name} -> [Ticket${ticketId} 현재 보유량=${ticket.stock} & 구매 요청량=${amount}]")
        ticket.purchase(amount)
        return ticketRepository.saveAndFlush(ticket)
    }
}
```

#### With K6

먼저 RDB에 티켓 1000장을 넣어두고 K6를 통해서 200명의 사용자가 동시에 티켓 5장씩 구매한 결과를 살펴보자

<div style="text-align: left">
  <img src="/assets/img/posts/2023-04-04-synchronized%20키워드를%20활용한%20동시성%20문제%20해결%20&%20한계/img17.png" alt="img"/>
</div>

```javascript
// K6 script
import http from "k6/http";

export const options = {
  scenarios: {
    spike: {
      executor: "constant-vus",
      vus: 200,
      duration: "1s",
      gracefulStop: "5m"
    },
  },
};

export default function () {
  const data = {
    "amount": 5
  }
  const res = http.post("http://localhost/api/v1/tickets/1/purchase", JSON.stringify(data), {
    headers: {
      "Content-Type": "application/json"
    },
  });
  console.log(res.body);
};
```

<div style="text-align: left">
  <img src="/assets/img/posts/2023-04-04-synchronized%20키워드를%20활용한%20동시성%20문제%20해결%20&%20한계/img18.png" alt="img"/>
</div>
<div style="text-align: left">
  <img src="/assets/img/posts/2023-04-04-synchronized%20키워드를%20활용한%20동시성%20문제%20해결%20&%20한계/img19.png" alt="img"/>
</div>

- 결과로 알 수 있듯이 `분산 환경`에서는 synchronized로 동시성 처리가 되지 않는다

### 분산환경에서 동시성 처리 방법

분산 환경에서 Race Condition에 대한 Mutual Exclusion을 보장하기 위해서는 다음 조건을 만족해야 한다

- N대의 서버가 `공통적으로 바라보는 공간`에서 락이라는 개념을 관리

<br>
위의 케이스에서는 아래와 같은 방법을 적용할 수 있다

1. Ticket Record에 직접적인 Lock을 적용해서 관리 (Pessimistic Lock)
2. Application 레벨에서 Version을 통해서 갱신 시점에 동기화를 관리 (Optimistic Lock)
3. DB Record가 아닌 별도의 외부 영역에서 락이라는 개념을 관리 (분산락 - MySQL Named Lock, Redis, Zookeeper, ...)

<br>
`1) Pessimistic Lock`을 활용해서 실제 동시성 처리가 이루어지는지 간단하게 테스트해보자

```kotlin
@PostMapping("/api/v2/tickets/{ticketId}/purchase")
fun purchaseV2(
    @PathVariable ticketId: Long,
    @RequestBody request: Request,
): Response {
    val ticket = serviceV6.purchase(ticketId, request.amount)
    return Response(
        server = getServer(),
        ticket = ticket,
    )
}

@Service
class TicketV6Service(
    private val ticketRepository: TicketRepository,
) {
    private val log: Logger = logger()

    @Transactional
    fun purchase(
        ticketId: Long,
        amount: Int,
    ): Ticket {
        val ticket = ticketRepository.findByIdWithLock(ticketId)
            ?: throw RuntimeException("Ticket not found ... $ticketId")
        log.info("${Thread.currentThread().name} -> [Ticket${ticketId} 현재 보유량=${ticket.stock} & 구매 요청량=${amount}]")
        ticket.purchase(amount)
        return ticket
    }
}

interface TicketRepository : JpaRepository<Ticket, Long> {
    @Query("SELECT t FROM Ticket t WHERE t.id = :id")
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    fun findByIdWithLock(@Param("id") id: Long): Ticket?
}
```

<div style="text-align: left">
  <img src="/assets/img/posts/2023-04-04-synchronized%20키워드를%20활용한%20동시성%20문제%20해결%20&%20한계/img20.png" alt="img"/>
</div>
<div style="text-align: left">
  <img src="/assets/img/posts/2023-04-04-synchronized%20키워드를%20활용한%20동시성%20문제%20해결%20&%20한계/img21.png" alt="img"/>
</div>

- Pessimistic Lock을 통해서 티켓 구매 동시성 제어에 성공하였다

<br>
> 관련된 코드는 [깃허브](https://github.com/sjiwon/devlog-codes/tree/main/spring/concurrency/ticket-concurrency-with-synchronized)에서 확인할 수 있습니다
