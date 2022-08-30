---
title: "Random values from PostgreSQL in intervals"
classes: wide
categories:
- programming
tags:
- kotlin
- postgresql
- jpa
---

As part of the search for optimal solutions, I've been looking for a way to get a different value from the database once a day with the lowest possible cost.
A first thought that was born was to move process of random selection to the database instead of application.

#### Versions
- `kotlin:1.6.21`
- `postgres:14.5`
- `hibernate:5.6.10`

### Procedure
```sql
create or replace function public.find_random_by_seed(seed float8) returns refcursor as $$ declare random_result refcursor;
begin
perform setseed(seed);

open random_result for 
select
random_value.id,
random_value.foo,
from
(
  select
    t.id, t.foo
from
test t offset floor(random() * ( select count(*) from test ))
limit 1 ) as random_value

return random_result;
end;

$$ language plpgsql;
```
This query could be shorter by using `order by random()`, but for performance reasons i chose `floor(random() * count(*))`

#### Execution plan for `floor(random() * count(*))` implementation

```sql
explain analyse 
select
random_value.id,
random_value.foo,
from
(
  select
    t.id, t.foo
from
test t offset floor(random() * ( select count(*) from test ))
limit 1 ) as random_value
	
Limit  (cost=2996.50..2996.52 rows=1 width=25) (actual time=14.053..14.054 rows=1 loops=1)
  InitPlan 1 (returns $0)
    ->  Aggregate  (cost=2746.81..2746.82 rows=1 width=8) (actual time=13.804..13.804 rows=1 loops=1)
          ->  Seq Scan on test  (cost=0.00..2496.85 rows=99985 width=0) (actual time=0.004..7.600 rows=100000 loops=1)
  ->  Seq Scan on test t  (cost=0.00..2496.85 rows=99985 width=25) (actual time=0.007..0.194 rows=1276 loops=1)
Planning Time: 0.080 ms
Execution Time: 14.072 ms
```

#### Execution plan for `order by random()` implementation

```sql
explain analyse select
	random_value.id as id,
	random_value.foo as foo
from
	(
	select
		t.id, t.foo
	from
		test t
	order by random()
	limit 1 ) as random_value
	
Subquery Scan on random_value  (cost=3246.74..3246.75 rows=1 width=25) (actual time=31.678..31.679 rows=1 loops=1)
  ->  Limit  (cost=3246.74..3246.74 rows=1 width=33) (actual time=31.676..31.677 rows=1 loops=1)
        ->  Sort  (cost=3246.74..3496.70 rows=99985 width=33) (actual time=31.674..31.674 rows=1 loops=1)
              Sort ty: (random())
              Sort Method: top-N heapsort  Memory: 25kB
              ->  Seq Scan on test t  (cost=0.00..2746.81 rows=99985 width=33) (actual time=0.018..16.048 rows=100000 loops=1)
Planning Time: 0.174 ms
Execution Time: 31.714 ms
```

As you can see the queries complexity differences are significant.

### Invocation

```kotlin
fun findRandomBySeed(seed: Double): List<Test> =
  with(entityManager.createStoredProcedureQuery("find_random_by_seed")) {
    registerStoredProcedureParameter(
      1,
      Void.TYPE,
      ParameterMode.REF_CURSOR
    )
    registerStoredProcedureParameter(2, Double::class.java, ParameterMode.IN)
    setParameter(2, seed)
    execute()

  return resultList.map { it as Array<*> }
    .map {
      Test(id = (it[0] as BigInteger).toLong(), foo = it[1] as String)
    }
}
```

```kotlin
val cache = AtomicReference(CachedTest(0L, Test(-1L, "")))

fun find(): Test {
    val currentDay = now(systemUTC())
    val seed = currentDay.toEpochDay()

    if (cache.seed() == seed) {
        return cache.test()
    }

    val test = testRepository.findRandomBySeed("0.$seed".toDouble())

    cache.getAndSet(CachedTest(seed = seed, test = test))
    return test
}

data class CachedTest(val seed: Long, val test: Test)

fun AtomicReference<CachedTest>.seed() = this.get().seed

fun AtomicReference<CachedTest>.test() = this.get().test
```

Advantage of this approach is that we don't have to additional schedule or caching functionality, procedure will be call once per day and only on first using (or after restart).