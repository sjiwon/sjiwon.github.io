---
title: 경매 입찰 & 작품 구매 동시성 문제 해결 2 - MySQL Named Lock
date: 2023-11-13 13:00 +0900
aliases: null
tags:
  - Spring
  - 동시성 처리
  - MySQL Named Lock
  - GET_LOCK 타임아웃
  - Lock 트랜잭션 분리
  - REQUIRES_NEW
image: /assets/img/thumbnails/Spring.png
categories:
  - Skill
  - Spring
---

## 개요

앞선 포스팅에서 [Pessimistic Write Lock](https://sjiwon.github.io/posts/%EA%B2%BD%EB%A7%A4-%EC%9E%85%EC%B0%B0-&-%EC%9E%91%ED%92%88-%EA%B5%AC%EB%A7%A4-%EB%8F%99%EC%8B%9C%EC%84%B1-%EB%AC%B8%EC%A0%9C-%ED%95%B4%EA%B2%B0-1-DB-Lock/)을 통해서 경매 입찰 & 작품 구매에 대한 동시성 문제를 해결하였다<br>
Pessimistic Write Lock은 `특정 DB Record에 Exclusive Lock`을 적용해서 다른 Thread들의 읽기/수정/삭제 모든 연산들을 대기시킴으로써 동시성을 제어한다

그런데 이렇게 DB Record 자체에 대한 Exclusive Lock을 통해서 제어하는 방식은 아래와 같은 문제를 발생시킬 수 있다

- 입찰 트랜잭션 & 구매 트랜잭션에 Exclusive Lock을 적용
- 관련된 auction, art, .. 테이블 Record에 대한 접근 제한
- 입찰 & 구매가 아닌 다른 로직 트랜잭션에서 auction, art, ...에 접근해야 하는 시점에 영향을 줄 수 있음
  - 이외 트랜잭션 Connection에서 해당 Record에 접근하지 못해서 전체적으로 처리가 밀리는 현상 발생 가능

<br>
따라서 이와 같이 DB Record에 직접적인 접근 제한을 거는것이 아니라 `외부 영역에서 Lock이라는 개념을 관리`하면 전체적인 처리를 효율적으로 진행할 수 있을거라고 판단된다

<br>

## MySQL Named Lock

|              함수              | 설명                                                    | 응답                                                                            |
|:----------------------------:|:------------------------------------------------------|:------------------------------------------------------------------------------|
| ***GET_LOCK(str, timeout)*** | - 문자열 str에 대한 lock 획득 시도<br>- 획득하지 못할 경우 timeout만큼 대기 | return 1 = Lock 획득 O<br>return 0 = timeout동안 Lock 획득 X<br>return null = 에러 발생 |
|   ***IS_FREE_LOCK(str)***    | 문자열 str에 대한 Lock을 획득할 수 있는 상태인지 확인                    | return 1 = 획득 가능<br>return 0 = 획득 불가능                                         |
|   ***IS_USED_LOCK(str)***    | 문자열 str에 대한 Lock이 사용중인지 여부                            | return connection identifier = 사용 중<br>return null = 사용 X                     |
|   ***RELEASE_LOCK(str)***    | 문자열 str에 대한 Lock 반납                                   | return 1 = Lock 반납 완료<br>return 0 = 반납할 Lock이 없는 경우                           |
|  ***RELEASE_ALL_LOCKS()***   | 모든 Lock 반납                                            | return x = 반납된 Lock 개수(x)                                                     |

### GET_LOCK(str, timeout)

GET_LOCK을 통해서 Named Lock을 얻는 경우 `해당 Lock은 베타적으로 동작`하기 때문에 다른 세션에서는 동일한 이름의 Lock을 획득할 수 없다<br>
GET_LOCK을 통해서 얻은 Named Lock은 `반드시 RELEASE를 호출함으로써 명시적으로 해제`해줘야 한다

- 트랜잭션 Commit/Rollback이 되었다고 Named Lock이 해제되지 않기 때문에 반드시 직접 해제해야 한다
- 만약 명시적으로 해제하지 않으면 계속 Lock을 물고 있어서 다른 트랜잭션에서 얻지 못하는 문제가 발생할 수 있다

<br>
그리고 다음과 같은 상황에서는 데드락이 발생할 여지도 존재한다

```sql
-- Session 1
SELECT GET_LOCK('sjiwon1', -1); 

-- Session 2
SELECT GET_LOCK('sjiwon2', -1); 

-- Session 1
SELECT GET_LOCK('sjiwon2', -1); -- Session 2가 release할때까지 wait 

-- Session 2
SELECT GET_LOCK('sjiwon1', -1); -- Session 1이 release할때까지 wait
```

<div style="text-align: left">
  <img src="/assets/img/posts/2023-11-13-경매%20입찰%20&%20작품%20구매%20동시성%20문제%20해결%202%20-%20MySQL%20Named%20Lock/img1.png" alt="img"/>
</div>
<div style="text-align: left">
  <img src="/assets/img/posts/2023-11-13-경매%20입찰%20&%20작품%20구매%20동시성%20문제%20해결%202%20-%20MySQL%20Named%20Lock/img2.png" alt="img"/>
</div>

### RELEASE_LOCK(str)

<div style="text-align: left">
  <img src="/assets/img/posts/2023-11-13-경매%20입찰%20&%20작품%20구매%20동시성%20문제%20해결%202%20-%20MySQL%20Named%20Lock/img3.png" alt="img"/>
</div>

- `return null`
  - str에 해당하는 Lock이 존재하지 않을 경우
- `return 0`
  - str에 해당하는 Lock이 존재하긴 하지만 ***<ins>해당 Session에서 획득한 Lock이 아닌 경우</ins>***
- `return 1`
  - 획득한 Lock을 성공적으로 반납

<br>

## MySQL Named Lock을 활용한 동시성 제어

```java
@Slf4j
@Component
@RequiredArgsConstructor
public class MySqlLockManager implements LockManager {
    private final NamedParameterJdbcTemplate jdbcTemplate;

    @Override
    public void acquire(final String key, final int timeout) {
//        final Map<String, Object> params = new HashMap<>();
//        params.put("key", key);
//        params.put("timeout", timeout);
//
//        final Integer result = jdbcTemplate.queryForObject(
//                "SELECT GET_LOCK(:key, :timeout)",
//                params,
//                Integer.class
//        );
//        checkResult(result, key, QueryType.GET_LOCK);

        final Integer result = jdbcTemplate.getJdbcTemplate().query(con -> {
                    log.info(">> GET_LOCK [{}] -> Connection = [{}] || Key = [{}] || Timeout = [{}]", Thread.currentThread().getName(), con, key, timeout);
                    final PreparedStatement preparedStatement = con.prepareStatement("SELECT GET_LOCK(?, ?)");
                    preparedStatement.setString(1, key);
                    preparedStatement.setInt(2, timeout);
                    return preparedStatement;
                }, rs -> {
                    if (rs.next()) {
                        return rs.getInt(1);
                    }
                    return null;
                }
        );
        checkResult(result, key, QueryType.GET_LOCK);
    }

    @Override
    public void release(final String key) {
//        final Map<String, Object> params = new HashMap<>();
//        params.put("key", key);
//
//        final Integer result = jdbcTemplate.queryForObject(
//                "SELECT RELEASE_LOCK(:key)",
//                params,
//                Integer.class
//        );
//        checkResult(result, key, QueryType.RELEASE_LOCK);

        final Integer result = jdbcTemplate.getJdbcTemplate().query(con -> {
                    log.info(">> RELEASE_LOCK [{}] -> Connection = [{}] || Key = [{}]", Thread.currentThread().getName(), con, key);
                    final PreparedStatement preparedStatement = con.prepareStatement("SELECT RELEASE_LOCK(?)");
                    preparedStatement.setString(1, key);
                    return preparedStatement;
                }, rs -> {
                    if (rs.next()) {
                        return rs.getInt(1);
                    }
                    return null;
                }
        );
        checkResult(result, key, QueryType.RELEASE_LOCK);
    }

    private void checkResult(final Integer result, final String key, final QueryType type) {
        if (result == null) {
            log.error("Named Lock 쿼리 결과가 null입니다 -> type = [{}], key = [{}]", type, key);
            throw new RuntimeException("Named lock result is null...");
        }
        if (result != 1) {
            log.error("Named Lock 쿼리 결과가 1(성공)이 아닙니다 -> type = [{}], result = [{}] key = [{}]", type, result, key);
            throw new RuntimeException("Named lock failed...");
        }
    }

    private enum QueryType {
        GET_LOCK, RELEASE_LOCK
    }
}
```

- Connection 로그 확인하기 위해서 getJdbcTemplate().query()를 통해서 쿼리를 작성하였다

```java
@Component
@RequiredArgsConstructor
public class BidFacade {
    private final LockManager lockManager;
    private final BidUseCase target;
 
    @Transactional
    public void invoke(final BidCommand command) {
        final String key = "AUCTION:" + command.auctionId();
        final int timeout = 5;
 
        try {
            lockManager.acquire(key, timeout);
            target.invoke(command);
        } finally {
            lockManager.release(key);
        }
    }
}

@UseCase
@RequiredArgsConstructor
public class BidUseCase {
    private final AuctionReader auctionReader;
    private final ArtReader artReader;
    private final MemberReader memberReader;
    private final BidInspector bidInspector;
    private final BidProcessor bidProcessor;
 
    @Transactional
    public void invoke(final BidCommand command) {
        final Auction auction = auctionReader.getById(command.auctionId());
        final Art art = artReader.getById(auction.getArtId());
        final Member bidder = memberReader.getById(command.memberId());
 
        bidInspector.checkBidCanBeProceed(auction, art, bidder, command.bidPrice());
        bidProcessor.execute(auction, bidder, command.bidPrice());
    }
}
```

> BidFacade#invoke에 ***@Transactional***을 적용한 이유는 다음과 같다<br>
> -> `Lock 획득 & 반납과 관련된 Connection을 Transaction별로 일치`시키기 위해서
{: .prompt-info }

<br>
결과를 살펴보자
<div style="text-align: left">
  <img src="/assets/img/posts/2023-11-13-경매%20입찰%20&%20작품%20구매%20동시성%20문제%20해결%202%20-%20MySQL%20Named%20Lock/img4.png" alt="img"/>
</div>

- 일단 Thread's Transaction별로 Lock 획득 & 반납 Connection은 일치함을 확인할 수 있다

<div style="text-align: left">
  <img src="/assets/img/posts/2023-11-13-경매%20입찰%20&%20작품%20구매%20동시성%20문제%20해결%202%20-%20MySQL%20Named%20Lock/img5.png" alt="img"/>
</div>
<div style="text-align: left">
  <img src="/assets/img/posts/2023-11-13-경매%20입찰%20&%20작품%20구매%20동시성%20문제%20해결%202%20-%20MySQL%20Named%20Lock/img6.png" alt="img"/>
</div>
<div style="text-align: left">
  <img src="/assets/img/posts/2023-11-13-경매%20입찰%20&%20작품%20구매%20동시성%20문제%20해결%202%20-%20MySQL%20Named%20Lock/img7.png" alt="img"/>
</div>

- 하지만 동시성 처리는 실패하였다

### 1. 중복 트랜잭션으로 인한 Connection 획득/반납 & 비즈니스 로직의 엉킴

위에서 실패한 원인을 아래 플로우를 통해서 알아보자

<div style="text-align: left">
  <img src="/assets/img/posts/2023-11-13-경매%20입찰%20&%20작품%20구매%20동시성%20문제%20해결%202%20-%20MySQL%20Named%20Lock/img8.png" alt="img"/>
</div>

이렇게 Lock을 얻고 반납하는 Transaction & 비즈니스 로직 Transaction이 엉킨 이유는 간단하다

> Lock을 얻고 반납하기 위한 Transaction에 비즈니스 로직 Transaction이 `참여(Default = REQUIRED)`<br>
> -> 그렇기 때문에 Lock을 완전히 반납하고 난 후에 다른 Transaction이 Lock을 얻어서 진행한다는 보장이 없는 것이다

<br>
해결방법 또한 간단하다

- Transaction을 분리하자

```java
@Component
@RequiredArgsConstructor
public class BidFacade {
    private final LockManager lockManager;
    private final BidUseCase target;
 
    @Transactional
    public void invoke(final BidCommand command) {
        final String key = "AUCTION:" + command.auctionId();
        final int timeout = 5;
 
        try {
            lockManager.acquire(key, timeout);
            target.invoke(command);
        } finally {
            lockManager.release(key);
        }
    }
}

@UseCase
@RequiredArgsConstructor
public class BidUseCase {
    private final AuctionReader auctionReader;
    private final ArtReader artReader;
    private final MemberReader memberReader;
    private final BidInspector bidInspector;
    private final BidProcessor bidProcessor;
 
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void invoke(final BidCommand command) {
        final Auction auction = auctionReader.getById(command.auctionId());
        final Art art = artReader.getById(auction.getArtId());
        final Member bidder = memberReader.getById(command.memberId());
 
        bidInspector.checkBidCanBeProceed(auction, art, bidder, command.bidPrice());
        bidProcessor.execute(auction, bidder, command.bidPrice());
    }
}
```

- `비즈니스 로직 Transaction의 propagation을 REQUIRES_NEW로 설정`함에 따라 Lock과 관련된 트랜잭션 & 비즈니스 로직 Transaction을 `분리`할 수 있다

### 2. GET_LOCK Timeout

위에서 Transaction을 분리하고 다시 테스트를 진행하였더니 다음과 같은 로그가 보인다

<div style="text-align: left">
  <img src="/assets/img/posts/2023-11-13-경매%20입찰%20&%20작품%20구매%20동시성%20문제%20해결%202%20-%20MySQL%20Named%20Lock/img9.png" alt="img"/>
</div>

GET_LOCK의 result가 0이라는 것은 무엇을 의미하는걸까?

- Timeout동안 기다려도 Lock을 획득하지 못했다는 의미이다

<br>
그러면 왜 Timeout이 지날동안 Lock을 획득하지 못했을까?

<div style="text-align: left">
  <img src="/assets/img/posts/2023-11-13-경매%20입찰%20&%20작품%20구매%20동시성%20문제%20해결%202%20-%20MySQL%20Named%20Lock/img10.png" alt="img"/>
</div>

그것은 바로 입찰 로직 진입점에 존재하는 BidFacade#invoke의 `@Transactional의 동작 원리 + HikariCP의 기본 Connection Pool` 때문이다

<br>
현재 application.yml에는 DBCP에 대한 어떠한 Connection Pool 설정이 없는 상태이다

<div style="text-align: left">
  <img src="/assets/img/posts/2023-11-13-경매%20입찰%20&%20작품%20구매%20동시성%20문제%20해결%202%20-%20MySQL%20Named%20Lock/img11.png" alt="img"/>
</div>

HikariCP는 설정값에 의해서 기본적인 Connection Pool Size를 10개로 잡아놓는다

<br>
이제 @Transactional에서 어떠한 일이 벌어지는지 간략하게 알아보자

<div style="text-align: left">
  <img src="/assets/img/posts/2023-11-13-경매%20입찰%20&%20작품%20구매%20동시성%20문제%20해결%202%20-%20MySQL%20Named%20Lock/img12.png" alt="img"/>
</div>

- Spring은 Transaction 관리에 대해서 DB Access Technique에 종속적이지 않게 하기 위해서 추상화한 메커니즘인 AbstractPlatformTransactionManager를 제공한다
- getTransaction을 통해서 새로운 Transaction을 생성하거나 기존 Transaction에 참여한다

<div style="text-align: left">
  <img src="/assets/img/posts/2023-11-13-경매%20입찰%20&%20작품%20구매%20동시성%20문제%20해결%202%20-%20MySQL%20Named%20Lock/img13.png" alt="img"/>
</div>

- doGetTransaction에서는 현재 JPA를 사용하고 있기 때문에 `EntityManager & Connection을 Holding하는 컴포넌트`를 return한다

> 자세한 내용은 [해당 포스팅](https://sjiwon.github.io/posts/Transaction-%EB%8F%99%EA%B8%B0%ED%99%94/)을 참고

<br>
여기까지 정리해보면 Transaction을 시작하게 되면 사전 준비 과정에서 `Connection을 얻는 구조`를 확인하였다

- 그렇다면 BidFacade에서 Transaction을 호출하고 invoke되는 BidUseCase에서는 REQUIRES_NEW로 새로운 Transaction을 열어버리는데 여기서 또 Connection을 획득할까?
- 정답

<br>
이제 Connection Pool과 현재 동시에 실행되는 Thread의 관계를 알아봐야 한다

- 위에서 봤듯이 yml에는 어떠한 Connection Pool 설정을 하지 않았기 때문에 maximumPoolSize는 10으로 설정되어 있다
- 그리고 테스트에서 동시에 실행되는 Thread는 10개이다

> 10개의 Thread가 동시에 BidFacade에 접근해서 Transaction을 시작하기 때문에 10개의 Connection을 소모하게 된다<br>
> 그 이후 Lock을 얻은 Thread가 BidUseCase의 로직에 접근하게 되는데 이 때 REQUIRES_NEW에 의해서 새로운 Transaction을 시작하게 되고 그에 따라서 새로운 Connection을 얻어야 한다<br>
> - 하지만 maximumPoolSize = 10이고 이미 다 소모했기 때문에 대기해야 한다
> - 이 PoolSize가 반납되기 위해서는 앞선 나머지 Thread들이 반납해줘야 한다
> - 그런데 또 나머지 Thread들은 Connection을 잡은 상태에서 Lock을 대기하게 된다
{: .prompt-tip }

따라서 이러한 상황에서 `추가적인 Connection을 얻기 위해서 계속 대기`하게 되고 결국 Timeout이 발생하는 것이다

- 결국 Deadlock으로 인해 서로의 자원을 계속 요구하고 있고 시간이 지남에 따라 Timeout 발생

<br>
이 상황에서 해결책은 2가지로 추릴 수 있을거 같다

1. Connection MaximumPoolSize 늘리기
2. Lock 획득/반납 & 비즈니스 로직에 대한 Connection을 아예 별개의 DataSource로 적용

Case 1로 해결할 경우 다음과 같은 문제가 발생할 수 있다<br>
maximumPoolSize를 20으로 늘려보자

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 20
```

<div style="text-align: left">
  <img src="/assets/img/posts/2023-11-13-경매%20입찰%20&%20작품%20구매%20동시성%20문제%20해결%202%20-%20MySQL%20Named%20Lock/img14.png" alt="img"/>
</div>

동시에 실행되는 Thread가 10개이고 DBCP를 20으로 늘렸더니 테스트가 성공하였다

- 동시에 실행되는 Thread를 20개로 늘리면? = 실패
- DBCP 30으로 늘리면? = 이제는 성공
- 동시에 실행되는 Thread를 30개로 늘리면? = 실패
- ...

결국 이러한 관계속에서 굉장히 세밀한 Thread, DBCP 수를 조절하는 것은 꽤 힘든일이다<br>
그리고 이러한 Lock을 얻는것 뿐만 아니라 아예 다른 로직에서 Connection이 필요한 경우까지 고려해야 하기 때문에 쉽지 않은 해결책이라고 생각하고 적절한 PoolSize를 고려하지 않으면 Timeout이 자주 발생할 가능성이 존재한다

<br>
따라서 위의 GET_LOCK Timeout과 관련된 해결책은 `아예 DataSource를 분리`함으로써 Lock과 관련없는 로직은 영향을 미치지 않도록 설계하는 것이 좋아보인다

<br>

## Pessimistic Lock vs MySQL Named Lock

X-Lock으로도 동시성 문제를 해결해보았고 MySQL Named Lock을 통해서도 동시성 문제를 해결해보았다
이 두가지 해결책의 가장 큰 차이점은 `DB Record에 어떠한 영향을 미치느냐`라고 생각한다

- `X-Lock`: Record 자체에 X-Lock을 걸기 때문에 다른 로직에서 해당 Record에 대한 Lock이 필요한 경우 영향을 미칠 수 있다
- `Named Lock`: Record가 아닌 외부 영역에서 관리하기 때문에 Record에 대한 다른 로직에서의 접근이 자유롭다

<br>
X-Lock의 경우 개요에서도 말했듯이 DB Record 자체에 Lock을 걸음으로써 다른 Task의 Connection에서 Lock이 필요한 상황에서 영향을 미칠 수 있다<br>
하지만 Named Lock의 경우 Record가 아닌 단순 문자열을 통해서 외부 영역에서 Lock을 관리하기 때문에 DB Record에 영향을 미치지 않고 안전하게 처리할 수 있다

- 물론 Lock과 관련된 Connection을 적절하게 관리함으로써 Connection을 얻는 것 자체적으로 다른 로직에 영향을 미치지 않도록 설계하는 것이 중요해보인다
