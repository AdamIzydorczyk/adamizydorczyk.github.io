---
title: "PostgreSQL DENSE_RANK for pagination and tempting performance trap"
classes: wide
categories:
- programming
tags:
- postgresql
- jooq
- sql
---

I looked for some simple solution to implements pagination for queries with joins. I had problem with typical limit offset approach, because it was really uncomfortable in case of dynamically created queries to keep part of query to pagination in subquery.

---

#### Versions

- `postgres:14.5`
- `jooq:7.1.1`
- `kotlin:1.6.21`

---

### Typical limit + offset implementation

```sql
select f.id, f.test, b.some_field from 
(select id, test 
from foo 
limit ?
offset ?) f
join bar b on f.id = b.foo_id
```

Equivalent of that simple query in JOOQ, looks that way.

```kotlin
val subSelect = select(FOO.ID, FOO.TEST)
  .from(FOO)
  .offset(pageStart)
  .limit(limit).asTable()

dsl.select(subSelect.field("id", FOO.ID.dataType),
  subSelect.field("test", FOO.TEST.dataType),
  BAR.SOME_FIELD)
  .from(subSelect)
  .join(BAR)
  .on(subSelect.field("id", FOO.ID.dataType)?.eq(BAR.FOO_ID))
```

As can be seen JOOQ implementation looks really complex for that simple query and it is relatively hard to dynamically extend.

### DENSE_RANK as a cure?

During looking for some simpler solution I found function that I didn't know before and that seemed very promising.

```sql
select * 
from (select dense_rank() over (order by f.id), f.id, f.test, b.definition 
from foo f
join bar b on b.foo_id = f.id
) t
```

```text
dense_rank	id	test	some_field
1	          1	  abc	  123
2	          2	  xyz	  444
2	          2	  xyz	  666
3	          3	  fgh	  876
```

In theory, just add filtering by ranges and problem should be solved.

```kotlin
dsl.select(asterisk())
  .from(select(
    denseRank().over(orderBy(FOO.ID)).`as`("dense_rank"),
    FOO.ID, 
    FOO.TEST, 
    BAR.SOME_FIELD)
    .from(FOO)
    .join(BAR)
    .on(FOO.ID.eq(BAR.FOO_ID)))
  .where(field(name("dense_rank")).between(pageStart, pageEnd))
```

The same result, much shorter code, on first impression cake eaten and owned :)

Unluckily that solution have one important problem. Filtering is evaluated after subquery execution, so database firstly collecting all data without pagination, and limiting result after that.

In other words, doesn't matter that we choose one or all elements in selected tables, query performance will be the same as without pagination.

##### 1 element pagination

```sql
explain analyse select * 
from (select dense_rank() over (order by f.id), f.id, f.test, b.some_field 
from foo f
join bar b on b.foo_id = f.id
) t
where t.dense_rank between 1 and 1

Subquery Scan on t  (cost=0.57..2345.91 rows=75 width=48) (actual time=0.080..57.682 rows=1 loops=1)
  Filter: ((t.dense_rank >= 1) AND (t.dense_rank <= 1))
  Rows Removed by Filter: 14978
  ->  WindowAgg  (cost=0.57..2121.39 rows=14968 width=48) (actual time=0.077..53.120 rows=14979 loops=1)
        ->  Merge Join  (cost=0.57..1896.87 rows=14968 width=40) (actual time=0.055..24.111 rows=14979 loops=1)
              Merge Cond: (f.id = b.foo_id)
              ->  Index Scan using foo_pk_unique_index on foo f  (cost=0.29..1135.64 rows=9965 width=24) (actual time=0.023..4.816 rows=10000 loops=1)
              ->  Index Scan using bar_foo_id_index on bar b  (cost=0.29..554.99 rows=14968 width=24) (actual time=0.022..6.184 rows=14979 loops=1)
Planning Time: 0.812 ms
Execution Time: 57.743 ms
```

##### 10000 elements pagination

```sql
explain analyse select * 
from (select dense_rank() over (order by f.id), f.id, f.test, b.some_field 
from foo f
join bar b on b.foo_id = f.id
) t
where t.dense_rank between 1 and 10000

Subquery Scan on t  (cost=0.57..2345.91 rows=75 width=48) (actual time=0.097..56.456 rows=14979 loops=1)
  Filter: ((t.dense_rank >= 1) AND (t.dense_rank <= 10000))
  ->  WindowAgg  (cost=0.57..2121.39 rows=14968 width=48) (actual time=0.094..50.342 rows=14979 loops=1)
        ->  Merge Join  (cost=0.57..1896.87 rows=14968 width=40) (actual time=0.067..22.899 rows=14979 loops=1)
              Merge Cond: (f.id = b.foo_id)
              ->  Index Scan using foo_pk_unique_index on foo f  (cost=0.29..1135.64 rows=9965 width=24) (actual time=0.029..4.689 rows=10000 loops=1)
              ->  Index Scan using bar_foo_id_index on bar b  (cost=0.29..554.99 rows=14968 width=24) (actual time=0.027..5.830 rows=14979 loops=1)
Planning Time: 0.647 ms
Execution Time: 58.032 ms
```

Results are identical for one and 10000 elements.
It is some example, that not every elegant solutions is optimal (most often they aren't).