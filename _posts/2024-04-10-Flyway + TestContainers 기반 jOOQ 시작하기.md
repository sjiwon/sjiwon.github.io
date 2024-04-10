---
title: Flyway + TestContainers 기반 jOOQ 시작하기
date: 2024-04-10 10:00 +0900
aliases: null
tags:
  - jOOQ
  - Flyway
  - TestContainers
  - Gradle jOOQ
  - Kotlin jOOQ
  - Kotlin QueryDsl
  - Kotlin JDSL
image: /assets/img/thumbnails/jOOQ.png
categories:
  - Skill
  - jOOQ
---

## jOOQ? QueryDsl?

> The easiest way to write SQL in Java

- [jOOQ](https://www.jooq.org/)
- [jOOQ Learn](https://www.jooq.org/learn/)

Spring 기반 웹 애플리케이션을 개발할 때 거의 대부분 `JPA (Java Persistence API)`라는 ORM을 선택한다<br>
JPA를 활용해서 개발을 하다가 DSL을 통해서 동적 쿼리를 편리하게 작성하기 위해서 `QueryDsl-JPA`을 도입한다

### QueryDsl에 대한 생각

QueryDsl은 <span style="color:red">Type-Safe</span>하게 JPQL을 작성할 수 있고 BooleanExpression을 통해서 Optional한 필드 쿼리 조건을 명시할 수 있다는
장점이 있다<br>
하지만 QueryDsl도 <span style="color:red">결국 JPQL Builder</span>이기 때문에 JPQL의 한계를 극복할 수는 없다

1. 성능 
   - 결국 DB로 질의를 하는 순간에는 JPQL -> SQL로 변환되어야 한다 
   - 이러한 변환 과정 자체가 Native SQL에 비해 성능 저하로 이어질 수 있다
2. Vendor Specific Functions 
   - DB Vendor별로 존재하는 Specific Function들을 Dialect 구현체마다 등록해서 사용한다는 점 자체가 번거롭다
3. From SubQuery 
   - Hibernate에서는 [Version 6.1](https://in.relation.to/2022/06/24/hibernate-orm-61-features/)부터 From절 SubQuery를 지원한다 
   - 하지만 QueryDsl은 오픈소스 업데이트가 적극적으로 진행되지 않고 있고 그에 따라서 From절 SubQuery는 아직 지원하지 않는것으로 파악된다

<br>
가장 핵심적으로 살펴볼 점은 QueryDsl이 사실상 방치된 오픈소스라는 것이다

- [Has QueryDSL started maintenance?](https://github.com/OpenFeign/querydsl/issues/277)
- [Attempt to reproduce problem reported by spring-data team](https://github.com/spring-projects/spring-data-jpa/pull/3346)

<br>
[QueryDsl Fork - OpenFeign](https://github.com/OpenFeign/querydsl)

- OpenFeign측에서 QueryDsl을 Fork하고 작업을 이어나가고 있고 Origin QueryDsl측에서도 2024/01/30 쯤에 `5.1.0` 버전을 relase했다
- 하지만 거의 3년동안 방치한 프로젝트가 다시 활발하게 활성화될지는 미지수이다

그렇다고 해서 지금 당장 QueryDsl을 버려라? 이 의미는 아니고 써도 된다

- [김영한 강사님 QueryDsl 의견](https://www.inflearn.com/questions/967391/%ED%98%84%EC%8B%9C%EC%A0%90-querydsl%EC%97%90-%EB%8C%80%ED%95%9C-%EC%9D%98%EA%B2%AC%EC%9D%B4-%EA%B6%81%EA%B8%88%ED%95%A9%EB%8B%88%EB%8B%A4)
- 위에서 여러 단점을 말했지만 필자도 QueryDsl을 활용해서 여러 동적 쿼리를 편리하게 작성해왔고 DSL로 작성하는 것에 대한 편리함 또한 인정한다

<br>
QueryDsl이 적극적으로 Release하지 않는것에 대해서 걱정이 있고 MyBatis에 질려서 Native SQL을 코드 레벨에서 DSL 방식으로 작성하는 것에 흥미가 있는 개발자라면 jOOQ도 굉장히 좋은 대안이 될 수 있다고 생각한다

- 본 포스팅에서는 jOOQ를 활용한 Query Build뿐만 아니라 `QueryDsl & Kotlin-JDSL`을 활용한 간단한 예제 또한 보여줄 예정이다

<br>

## 실습

> 관련된 코드는 [깃허브](https://github.com/sjiwon/devlog-codes/tree/main/spring/jooq/start-with-flyway-testcontainers)에서 확인할 수
> 있습니다

### 1. 도메인 정의 & ERD

굉장히 심플한 도메인 관계를 정의할 것이다

1. 배우(Actor) & 영화(Film)
2. 배우는 여러 영화에 출연할 수 있다
3. 영화에는 여러 배우가 출연할 수 있다

```sql
CREATE TABLE IF NOT EXISTS actor
(
    `id`   BIGINT AUTO_INCREMENT PRIMARY KEY,
    `name` VARCHAR(100) NOT NULL
) ENGINE = InnoDB
  CHARSET = UTF8MB4;
  
CREATE TABLE IF NOT EXISTS film
(
    `id`           BIGINT AUTO_INCREMENT PRIMARY KEY,
    `title`        VARCHAR(100) NOT NULL,
    `description`  VARCHAR(100) NOT NULL,
    `release_date` DATE         NOT NULL,
    `length`       INT          NOT NULL
) ENGINE = InnoDB
  CHARSET = UTF8MB4;

CREATE TABLE IF NOT EXISTS film_actor
(
    `id`       BIGINT AUTO_INCREMENT PRIMARY KEY,
    `actor_id` BIGINT       NOT NULL,
    `film_id`  BIGINT       NOT NULL,
    `role`     VARCHAR(255) NOT NULL
) ENGINE = InnoDB
  CHARSET = UTF8MB4;

ALTER TABLE film_actor
    ADD CONSTRAINT fk_film_actor_actor_id_from_actor
        FOREIGN KEY (actor_id)
            REFERENCES actor (id);

ALTER TABLE film_actor
    ADD CONSTRAINT fk_film_actor_film_id_from_film
        FOREIGN KEY (film_id)
            REFERENCES film (id);
```

<div style="text-align: left">
  <img src="/assets/img/posts/2024-04-10-Flyway%20+%20TestContainers%20기반%20jOOQ%20시작하기/img1.png" alt="img"/>
</div>

### 2. Gradle

Gradle 기반 `jOOQ & QueryDsl Model`을 생성해보자

```kotlin
import nu.studer.gradle.jooq.JooqEdition
import nu.studer.gradle.jooq.JooqGenerate
import org.jooq.meta.jaxb.ForcedType
import org.jooq.meta.jaxb.Logging
import org.springframework.boot.gradle.tasks.bundling.BootJar
import org.testcontainers.containers.MySQLContainer
import org.testcontainers.utility.DockerImageName

plugins {
    ...
    id("org.flywaydb.flyway")
    id("nu.studer.jooq")
    kotlin("kapt")
    kotlin("plugin.jpa")
}

dependencies {
    // Data
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    runtimeOnly("com.mysql:mysql-connector-j:${property("mysqlVersion")}")
    implementation("org.flywaydb:flyway-core:${property("flywayVersion")}")
    implementation("org.flywaydb:flyway-mysql:${property("flywayVersion")}")

    // jOOQ
    implementation("org.springframework.boot:spring-boot-starter-jooq")
    jooqGenerator("mysql:mysql-connector-java:${property("mysqlVersion")}")
    jooqGenerator("org.jooq:jooq:${property("jooqVersion")}")
    jooqGenerator("org.jooq:jooq-codegen:${property("jooqVersion")}")
    jooqGenerator("org.jooq:jooq-meta:${property("jooqVersion")}")

    // QueryDsl
    implementation("com.querydsl:querydsl-jpa:${property("queryDslVersion")}:jakarta")
    kapt("com.querydsl:querydsl-apt:${property("queryDslVersion")}:jakarta")
    kapt("jakarta.annotation:jakarta.annotation-api")
    kapt("jakarta.persistence:jakarta.persistence-api")

    // Kotlin JDSL
    implementation("com.linecorp.kotlin-jdsl:jpql-dsl:${property("kotlinJdslVersion")}")
    implementation("com.linecorp.kotlin-jdsl:jpql-render:${property("kotlinJdslVersion")}")
    implementation("com.linecorp.kotlin-jdsl:spring-data-jpa-support:${property("kotlinJdslVersion")}")
}

// jOOQ
val mysqlContainer = tasks.register("mysqlContainer") {
    val container = MySQLContainer<Nothing>(DockerImageName.parse("mysql:8.0.33")).apply {
        withDatabaseName("dsl")
        withUsername("root")
        withPassword("1234")
        start()
    }

    extra.set("jdbcUrl", container.jdbcUrl)
    extra.set("username", container.username)
    extra.set("password", container.password)
    extra.set("databaseName", container.databaseName)
    extra.set("container", container)
}

val mysqlJdbcUrl = mysqlContainer.get().extra["jdbcUrl"].toString()
val mysqlUsername = mysqlContainer.get().extra["username"].toString()
val mysqlPassword = mysqlContainer.get().extra["password"].toString()
val mysqlDatabaseName = mysqlContainer.get().extra["databaseName"].toString()
val container = mysqlContainer.get().extra["container"] as MySQLContainer<*>

buildscript {
    dependencies {
        classpath("mysql:mysql-connector-java:${property("mysqlVersion")}")
        classpath("org.flywaydb:flyway-mysql:${property("flywayVersion")}")
        classpath("org.testcontainers:mysql:${property("testContainersVersion")}")
    }

    configurations["classpath"].resolutionStrategy.eachDependency {
        if (requested.group.startsWith("org.jooq") && requested.name.startsWith("jooq")) {
            useVersion("${property("jooqVersion")}")
        }
    }
}

flyway {
    locations = arrayOf("filesystem:src/main/resources/db/migration")
    url = mysqlJdbcUrl
    user = mysqlUsername
    password = mysqlPassword
}

jooq {
    version.set("${property("jooqVersion")}")
    edition.set(JooqEdition.OSS)

    configurations {
        create("main") {
            jooqConfiguration.apply {
                logging = Logging.WARN
                jdbc.apply {
                    driver = "com.mysql.cj.jdbc.Driver"
                    url = mysqlJdbcUrl
                    user = mysqlUsername
                    password = mysqlPassword
                }
                generator.apply {
                    name = "org.jooq.codegen.DefaultGenerator"
                    database.apply {
                        name = "org.jooq.meta.mysql.MySQLDatabase"
                        inputSchema = mysqlDatabaseName
                        forcedTypes.addAll(
                            arrayOf(
                                ForcedType()
                                    .withName("varchar")
                                    .withIncludeExpression(".*")
                                    .withIncludeTypes("JSONB?"),
                                ForcedType()
                                    .withName("varchar")
                                    .withIncludeExpression(".*")
                                    .withIncludeTypes("INET")
                            ).toList()
                        )
                    }
                    generate.apply {
                        isDaos = true
                        isDeprecated = false
                        isRecords = true
                        isImmutablePojos = true
                        isFluentSetters = true
                        isJavaTimeTypes = true
                    }
                    target.apply {
                        directory = "build/generated/jooq" // jOOQ Model 생성 위치
                        packageName = "com.sjiwon.jooq" // 패키지명
                        encoding = "UTF-8"
                    }
                    strategy.name = "org.jooq.codegen.example.JPrefixGeneratorStrategy" // jOOQ Model Prefix Strategy
                }
            }
        }
    }
}

tasks.named<JooqGenerate>("generateJooq") {
    dependsOn("mysqlContainer")
    dependsOn("flywayMigrate")

    inputs.files(fileTree("src/main/resources/db/migration"))
        .withPropertyName("migration")
        .withPathSensitivity(PathSensitivity.RELATIVE)

    allInputsDeclared.set(true)
    outputs.cacheIf { true }
    doLast { container.stop() }
}
```

<div style="text-align: left">
  <img src="/assets/img/posts/2024-04-10-Flyway%20+%20TestContainers%20기반%20jOOQ%20시작하기/img2.png" alt="img"/>
</div>

- <span style="color:red">Flyway + TestContainers</span> 기반으로 jOOQ Model을 생성하였다

### 3. 쿼리 작성

```kotlin
@Entity
@Table(name = "actor")
class Actor(
    @Id
    @GeneratedValue(strategy = IDENTITY)
    val id: Long = 0L,
    val name: String,
)

@Entity
@Table(name = "film")
class Film(
    @Id
    @GeneratedValue(strategy = IDENTITY)
    val id: Long = 0L,
    val title: String,
    val description: String,
    val releaseDate: LocalDate,
    val length: Int,
)

@Entity
@Table(name = "film_actor")
class FilmActor(
    @Id
    @GeneratedValue(strategy = IDENTITY)
    val id: Long = 0L,
    val actorId: Long,
    val filmId: Long,
    val role: String,
)
```

```kotlin
interface QueryRepository {
    /**
     * 특정 배우가 참여한 영화 정보
     */
    fun fetchParticipatedFilms(name: String): List<ParticipatedFilm>

    /**
     * 특정 영화에 참여한 배우 정보
     */
    fun fetchParticipatedActors(title: String): List<ParticipatedActor>
}

data class ParticipatedFilm @QueryProjection constructor(
    val id: Long,
    val title: String,
    val releaseDate: LocalDate,
    val role: String,
)

data class ParticipatedActor @QueryProjection constructor(
    val id: Long,
    val name: String,
    val role: String,
)
```

#### 1) jOOQ

```kotlin
@Repository
class JooqRepository(
    private val dsl: DSLContext,
) : QueryRepository {
    override fun fetchParticipatedFilms(name: String): List<ParticipatedFilm> {
        return dsl.select(
            FILM.ID,
            FILM.TITLE,
            FILM.RELEASE_DATE,
            FILM_ACTOR.ROLE,
        ).from(FILM_ACTOR)
            .innerJoin(ACTOR).on(ACTOR.ID.eq(FILM_ACTOR.ACTOR_ID))
            .innerJoin(FILM).on(FILM.ID.eq(FILM_ACTOR.FILM_ID))
            .where(ACTOR.NAME.eq(name))
            .orderBy(FILM.RELEASE_DATE.asc())
            .fetchInto(ParticipatedFilm::class.java)
    }

    override fun fetchParticipatedActors(title: String): List<ParticipatedActor> {
        return dsl.select(
            ACTOR.ID,
            ACTOR.NAME,
            FILM_ACTOR.ROLE,
        ).from(FILM_ACTOR)
            .innerJoin(ACTOR).on(ACTOR.ID.eq(FILM_ACTOR.ACTOR_ID))
            .innerJoin(FILM).on(FILM.ID.eq(FILM_ACTOR.FILM_ID))
            .where(FILM.TITLE.eq(title))
            .orderBy(ACTOR.NAME.asc())
            .fetchInto(ParticipatedActor::class.java)
    }
}
```

#### 2) QueryDsl

```kotlin
@Repository
class QueryDslRepository(
    private val query: JPAQueryFactory,
) : QueryRepository {
    override fun fetchParticipatedFilms(name: String): List<ParticipatedFilm> {
        return query.select(
            QParticipatedFilm(
                film.id,
                film.title,
                film.releaseDate,
                filmActor.role
            )
        ).from(filmActor)
            .innerJoin(actor).on(actor.id.eq(filmActor.actorId))
            .innerJoin(film).on(film.id.eq(filmActor.filmId))
            .where(actor.name.eq(name))
            .orderBy(film.releaseDate.asc())
            .fetch()
    }

    override fun fetchParticipatedActors(title: String): List<ParticipatedActor> {
        return query.select(
            QParticipatedActor(
                actor.id,
                actor.name,
                filmActor.role
            )
        ).from(filmActor)
            .innerJoin(actor).on(actor.id.eq(filmActor.actorId))
            .innerJoin(film).on(film.id.eq(filmActor.filmId))
            .where(film.title.eq(title))
            .orderBy(actor.name.asc())
            .fetch()
    }
}
```

#### 3) Kotlin JDSL

```kotlin
@Repository
class JdslRepository(
    private val em: EntityManager,
    private val context: JpqlRenderContext,
) : QueryRepository {
    override fun fetchParticipatedFilms(name: String): List<ParticipatedFilm> {
        val query: SelectQuery<ParticipatedFilm> = jpql {
            selectNew<ParticipatedFilm>(
                path(Film::id),
                path(Film::title),
                path(Film::releaseDate),
                path(FilmActor::role),
            ).from(
                entity(FilmActor::class),
                innerJoin(Actor::class).on(path(Actor::id).equal(path(FilmActor::actorId))),
                innerJoin(Film::class).on(path(Film::id).equal(path(FilmActor::filmId)))
            ).where(
                path(Actor::name).equal(name),
            ).orderBy(
                path(Film::releaseDate).asc(),
            )
        }
        return em.createQuery(query, context).resultList
    }

    override fun fetchParticipatedActors(title: String): List<ParticipatedActor> {
        val query: SelectQuery<ParticipatedActor> = jpql {
            selectNew<ParticipatedActor>(
                path(Actor::id),
                path(Actor::name),
                path(FilmActor::role),
            ).from(
                entity(FilmActor::class),
                innerJoin(Actor::class).on(path(Actor::id).equal(path(FilmActor::actorId))),
                innerJoin(Film::class).on(path(Film::id).equal(path(FilmActor::filmId)))
            ).where(
                path(Film::title).equal(title),
            ).orderBy(
                path(Actor::name).asc(),
            )
        }
        return em.createQuery(query, context).resultList
    }
}
```

- Kotlin JDSL은 QueryDsl & jOOQ처럼 APT(Annotation Processing Tool)나 CodeGenerator로 Model을 생성하는 것이 아니라 `KClass, KProperty`기반으로 DSL을 제공한다
- 굳이 단점을 꼽자면 Kotlin에서만 사용할 수 있고 기능 자체는 QueryDsl, jOOQ보다 부족하다

### 4. 쿼리 테스트

```kotlin
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
@ContextConfiguration(
    initializers = [MySqlTestContainers.Initializer::class],
)
@ExtendWith(DatabaseCleanerEachCallbackExtension::class)
@IntegrationTest
class QueryTest(
    private val actorRepository: ActorRepository,
    private val filmRepository: FilmRepository,
    private val filmActorRepository: FilmActorRepository,
    private val jooq: JooqRepository,
    private val queryDsl: QueryDslRepository,
    private val jdsl: JdslRepository,
) {
    private lateinit var actors: List<Actor>
    private lateinit var films: List<Film>

    @BeforeEach
    fun setUp() {
        actors = actorRepository.saveAll(
            listOf(Actor(name = "Actor1"), Actor(name = "Actor2"), Actor(name = "Actor3"))
        )
        films = filmRepository.saveAll(
            listOf(
                Film(title = "Film1", description = "Hello Film1", releaseDate = LocalDate.of(2023, 5, 1), length = 80),
                Film(title = "Film2", description = "Hello Film2", releaseDate = LocalDate.of(2018, 2, 10), length = 100),
                Film(title = "Film3", description = "Hello Film3", releaseDate = LocalDate.of(2020, 6, 18), length = 90),
            )
        )
        filmActorRepository.saveAll(
            listOf(
                FilmActor(actorId = actors[0].id, filmId = films[0].id, role = filmRole(films[0], actors[0])),
                FilmActor(actorId = actors[1].id, filmId = films[0].id, role = filmRole(films[0], actors[1])),
                FilmActor(actorId = actors[2].id, filmId = films[0].id, role = filmRole(films[0], actors[2])),

                FilmActor(actorId = actors[0].id, filmId = films[1].id, role = filmRole(films[1], actors[0])),
                FilmActor(actorId = actors[1].id, filmId = films[1].id, role = filmRole(films[1], actors[1])),

                FilmActor(actorId = actors[0].id, filmId = films[2].id, role = filmRole(films[2], actors[0])),
                FilmActor(actorId = actors[2].id, filmId = films[2].id, role = filmRole(films[2], actors[2])),
            )
        )
    }

    @Test
    fun `특정 배우가 참여한 영화 정보를 조회한다`() {
        val resultJooq1 = jooq.fetchParticipatedFilms(actors[0].name)
        val resultQueryDsl1 = queryDsl.fetchParticipatedFilms(actors[0].name)
        val resultJdsl1 = jdsl.fetchParticipatedFilms(actors[0].name)
        assertParticipatedFilmsMatch(
            result = listOf(resultJooq1, resultQueryDsl1, resultJdsl1),
            filmIds = listOf(films[1].id, films[2].id, films[0].id),
            actorRoles = listOf(filmRole(films[1], actors[0]), filmRole(films[2], actors[0]), filmRole(films[0], actors[0])),
        )

        val resultJooq2 = jooq.fetchParticipatedFilms(actors[1].name)
        val resultQueryDsl2 = queryDsl.fetchParticipatedFilms(actors[1].name)
        val resultJdsl2 = jdsl.fetchParticipatedFilms(actors[1].name)
        assertParticipatedFilmsMatch(
            result = listOf(resultJooq2, resultQueryDsl2, resultJdsl2),
            filmIds = listOf(films[1].id, films[0].id),
            actorRoles = listOf(filmRole(films[1], actors[1]), filmRole(films[0], actors[1])),
        )

        val resultJooq3 = jooq.fetchParticipatedFilms(actors[2].name)
        val resultQueryDsl3 = queryDsl.fetchParticipatedFilms(actors[2].name)
        val resultJdsl3 = jdsl.fetchParticipatedFilms(actors[2].name)
        assertParticipatedFilmsMatch(
            result = listOf(resultJooq3, resultQueryDsl3, resultJdsl3),
            filmIds = listOf(films[2].id, films[0].id),
            actorRoles = listOf(filmRole(films[2], actors[2]), filmRole(films[0], actors[2])),
        )
    }

    @Test
    fun `특정 영화에 참여한 배우 정보를 조회한다`() {
        val resultJooq1 = jooq.fetchParticipatedActors(films[0].title)
        val resultQueryDsl1 = queryDsl.fetchParticipatedActors(films[0].title)
        val resultJdsl1 = jdsl.fetchParticipatedActors(films[0].title)
        assertParticipatedActorsMatch(
            result = listOf(resultJooq1, resultQueryDsl1, resultJdsl1),
            actorIds = listOf(actors[0].id, actors[1].id, actors[2].id),
            actorRoles = listOf(filmRole(films[0], actors[0]), filmRole(films[0], actors[1]), filmRole(films[0], actors[2])),
        )

        val resultJooq2 = jooq.fetchParticipatedActors(films[1].title)
        val resultQueryDsl2 = queryDsl.fetchParticipatedActors(films[1].title)
        val resultJdsl2 = jdsl.fetchParticipatedActors(films[1].title)
        assertParticipatedActorsMatch(
            result = listOf(resultJooq2, resultQueryDsl2, resultJdsl2),
            actorIds = listOf(actors[0].id, actors[1].id),
            actorRoles = listOf(filmRole(films[1], actors[0]), filmRole(films[1], actors[1])),
        )

        val resultJooq3 = jooq.fetchParticipatedActors(films[2].title)
        val resultQueryDsl3 = queryDsl.fetchParticipatedActors(films[2].title)
        val resultJdsl3 = jdsl.fetchParticipatedActors(films[2].title)
        assertParticipatedActorsMatch(
            result = listOf(resultJooq3, resultQueryDsl3, resultJdsl3),
            actorIds = listOf(actors[0].id, actors[2].id),
            actorRoles = listOf(filmRole(films[2], actors[0]), filmRole(films[2], actors[2])),
        )
    }

    private fun filmRole(
        film: Film,
        actor: Actor,
    ): String = "File${film.id}-Role${actor.id}"

    private fun assertParticipatedFilmsMatch(
        result: List<List<ParticipatedFilm>>,
        filmIds: List<Long>,
        actorRoles: List<String>,
    ) {
        result.forEach { record ->
            record.map { it.id } shouldContainExactly filmIds
            record.map { it.role } shouldContainExactly actorRoles
        }
    }

    private fun assertParticipatedActorsMatch(
        result: List<List<ParticipatedActor>>,
        actorIds: List<Long>,
        actorRoles: List<String>,
    ) {
        result.forEach { record ->
            record.map { it.id } shouldContainExactly actorIds
            record.map { it.role } shouldContainExactly actorRoles
        }
    }
}
```

```sql
-- jOOQ
select
        `dsl`.`film`.`id`,
        `dsl`.`film`.`title`,
        `dsl`.`film`.`release_date`,
        `dsl`.`film_actor`.`role` 
    from
        `dsl`.`film_actor` 
    join
        `dsl`.`actor` 
            on `dsl`.`actor`.`id` = `dsl`.`film_actor`.`actor_id` 
    join
        `dsl`.`film` 
            on `dsl`.`film`.`id` = `dsl`.`film_actor`.`film_id` 
    where
        `dsl`.`actor`.`name` = 'Actor1' 
    order by
        `dsl`.`film`.`release_date` asc

-- QueryDsl
select
        f1_0.id,
        f1_0.title,
        f1_0.release_date,
        fa1_0.role 
    from
        film_actor fa1_0 
    join
        actor a1_0 
            on a1_0.id=fa1_0.actor_id 
    join
        film f1_0 
            on f1_0.id=fa1_0.film_id 
    where
        a1_0.name='Actor1' 
    order by
        f1_0.release_date

-- Kotlin JDSL
select
        f1_0.id,
        f1_0.title,
        f1_0.release_date,
        fa1_0.role 
    from
        film_actor fa1_0 
    join
        actor a1_0 
            on a1_0.id=fa1_0.actor_id 
    join
        film f1_0 
            on f1_0.id=fa1_0.film_id 
    where
        a1_0.name='Actor1' 
    order by
        f1_0.release_date
```

<div style="text-align: left">
  <img src="/assets/img/posts/2024-04-10-Flyway%20+%20TestContainers%20기반%20jOOQ%20시작하기/img3.png" alt="img"/>
</div>
