---
title: "SQL existing checking rediscovered"
classes: wide
categories:
- programming
tags:
- sql
- postgresql
- jpa
---

I found out, that by whole of my professional career I was doing something so simple like existing checking by sql in wrong way.

It usually looked like this with checking that result is greater than 0.

#### Example
```postgres
select count(t) from test t where t.foo = :foo 
```

For that query execution plan looks like this

```postgres
explain analyse select count(t) from test t where t.foo = :foo 

Aggregate  (cost=2746.82..2746.83 rows=1 width=8) (actual time=9.424..9.424 rows=1 loops=1)
  ->  Seq Scan on test t  (cost=0.00..2746.81 rows=1 width=112) (actual time=0.011..9.419 rows=1 loops=1)
        Filter: ((foo)::text = 'test'::text)
        Rows Removed by Filter: 99999
Planning Time: 0.056 ms
Execution Time: 9.442 ms
```
Unfortunately aggregation is relatively expensive operation for that use case, but by little improvement it can be done better.
#### Alternative
```postgres
explain analyse 1 from test t where t.foo = :foo limit 1

Limit  (cost=0.00..2746.81 rows=1 width=4) (actual time=0.008..0.008 rows=1 loops=1)
  ->  Seq Scan on test t  (cost=0.00..2746.81 rows=1 width=4) (actual time=0.007..0.007 rows=1 loops=1)
        Filter: ((foo)::text = 'test'::text)
        Rows Removed by Filter: 2
Planning Time: 0.043 ms
Execution Time: 0.017 ms
```
Now it's enough to check that query return some result.

Returning `1` instead of row is especially crucial in context of jpa queries, in construction like below.

```kotlin
entityManager.createQuery("select 1 from Test t where t.foo = :$TEST")
    .setParameter(TEST, "test")
    .setMaxResults(1)
    .resultList
    .isNotEmpty()
```

In case of using `select t from Test t` instead of `select 1 from Test t` jpa will create unnecessary object with assigned fields by query result.