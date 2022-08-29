---
title: "Adventure with Ratpack and PostgreSQL R2DBC - The Beginning"
classes: wide
categories:
- programming
tags:
- kotlin
- spring
- ratpack
- postgresql
- r2dbc
---

Encouraged by opinions about Ratpack library as lightweight alternative for typical Spring + Tomcat approach I decided to see what it is and how it compares with the standard implementation.

Additionally to tests instead of JDBC based connection i choosed R2DBC driver to PostgreSQL to make solution fully non blocking.

---

#### Versions
- `kotlin:1.6.21`
- `ratpack-core:1.9.0`
- `r2dbc-postgresql:0.8.12.RELEASE`
- `spring-boot-starter-data-jdbc:2.7.2`
- `spring-boot-starter-data-web:2.7.2`

### Implementation

#### Ratpack + R2DBC

```kotlin
fun fetchAll(): Flux<Test> = Flux.usingWhen(
    connectionFactory.create(),
    { c -
        c.createStatement("select t.id, t.word from test t order by random() limit 1000")
            .execute()
    },
    PostgresqlConnection::close)
.flatMap { result -
    result.map { row, _ -
        Test(
                id = row.get("id", Long::class.javaObjectType) ?: -1,
                word = row.get("test", String::class.java).toString())
    }
}
```

```kotlin
fun handleRequest(context: Context)
    = context.render(Jackson.chunkedJsonList(context, service.fetchAll()))
```

```kotlin
RatpackServer.start { spec: RatpackServerSpec -
    spec.handlers { chain: Chain -
        chain.path("test") { ctx: Context -
            ctx.byMethod { m: ByMethodSpec -
                m.get(TestHandler::handleRequest)
            }
        }
    }
}
```

#### Spring Boot + Tomcat + JDBC

```kotlin
@GetMapping
fun findAll(): List<Test> 
    = jdbcTemplate.query("select t.id, t.word from test t order by random() limit 1000")
            {rs, _ - Test(rs.getLong("id"), rs.getString("test"))}
```

#### Gatling

```scala
class TestSimulation extends Simulation {
    val httpProtocol: HttpProtocolBuilder = http
        .baseUrl(URL)
        .acceptHeader("application/json")

    val scn: ScenarioBuilder = scenario("test").repeat(REPEAT) {
        exec(http("test")
            .get("/test"))
    }

    setUp(scn.inject(atOnceUsers(USERS)).protocols(httpProtocol))
}
```

### Results - 1000 requests

#### Ratpack + R2DBC - 10 x 10 requests - 10k elements in table
```plaintext
================================================================================
---- Global Information --------------------------------------------------------
 request count                                        100 (OK=100    KO=0     )
 min response time                                     37 (OK=37     KO=-     )
 max response time                                    106 (OK=106    KO=-     )
 mean response time                                    67 (OK=67     KO=-     )
 std deviation                                         16 (OK=16     KO=-     )
 response time 50th percentile                         64 (OK=64     KO=-     )
 response time 75th percentile                         75 (OK=75     KO=-     )
 response time 95th percentile                         96 (OK=96     KO=-     )
 response time 99th percentile                         98 (OK=98     KO=-     )
 mean requests/sec                                    100 (OK=100    KO=-     )
================================================================================
```

#### Spring Boot + Tomcat + JDBC - 10 x 10 requests - 10k elements in table
```plaintext
================================================================================
---- Global Information --------------------------------------------------------
 request count                                        100 (OK=100    KO=0     )
 min response time                                     16 (OK=16     KO=-     )
 max response time                                     53 (OK=53     KO=-     )
 mean response time                                    28 (OK=28     KO=-     )
 std deviation                                          7 (OK=7      KO=-     )
 response time 50th percentile                         26 (OK=26     KO=-     )
 response time 75th percentile                         30 (OK=30     KO=-     )
 response time 95th percentile                         38 (OK=38     KO=-     )
 response time 99th percentile                         51 (OK=51     KO=-     )
 mean requests/sec                                    100 (OK=100    KO=-     )
================================================================================
```

The first results came out strongly disappointing, with default configuration standard implementation achieved better results.

Moreover on attempt with setting up more requests, began to appear exceptions `io.r2dbc.postgresql.ExceptionFactory$PostgresqlNonTransientResourceException: [53300] sorry, too many clients already`

Which was due to not reusing connection by default in any ways by r2dbc driver.

To fix it i added library `io.r2dbc:r2dbc-pool:0.9.1.RELEASE` that provides connection pool like indicate name :)

### Results - 10000 requests

#### Ratpack + R2DBC - 100 x 100 requests - 100k elements in table
```plaintext
================================================================================
---- Global Information --------------------------------------------------------
 request count                                      10000 (OK=10000  KO=0     )
 min response time                                     23 (OK=23     KO=-     )
 max response time                                  43158 (OK=43158  KO=-     )
 mean response time                                   417 (OK=417    KO=-     )
 std deviation                                       1357 (OK=1357   KO=-     )
 response time 50th percentile                        412 (OK=412    KO=-     )
 response time 75th percentile                        422 (OK=422    KO=-     )
 response time 95th percentile                        445 (OK=445    KO=-     )
 response time 99th percentile                        536 (OK=536    KO=-     )
 mean requests/sec                                208.333 (OK=208.333 KO=-    )
================================================================================
```

#### Spring Boot + Tomcat + JDBC - 100 x 100 requests - 100k elements in table
```plaintext
================================================================================
---- Global Information --------------------------------------------------------
 request count                                      10000 (OK=10000  KO=0     )
 min response time                                     24 (OK=24     KO=-     )
 max response time                                   1257 (OK=1257   KO=-     )
 mean response time                                   405 (OK=405    KO=-     )
 std deviation                                         87 (OK=87     KO=-     )
 response time 50th percentile                        412 (OK=412    KO=-     )
 response time 75th percentile                        445 (OK=445    KO=-     )
 response time 95th percentile                        468 (OK=468    KO=-     )
 response time 99th percentile                        736 (OK=736    KO=-     )
 mean requests/sec                                238.095 (OK=238.095 KO=-    )
================================================================================
```

For a larger number of requests the disproportion is still visible.

That is the closer result to standard implementation, with none of the attempts to change size of data set, number of requests or pools size I didn't reach better result for non blocking implementation.

I don't know how representative is my test, but in select operations this approach didn't give any profit, except radically lower memory consumption by ratpack with comparison to spring-web.

![JConsole Screen](/assets/images/2022-08-27-adventure-with-ratpack-and-postgresql-r2dbc-beginning.jpg)