---
title: "Complex entity field with Hibernate @Formula and PostgreSQL procedure"
classes: wide
categories:
- programming
tags:
- kotlin
- postgresql
- jpa
- hibernate
---

Formula annotation is very useful to add some additional fields by subqueries to our entities.
Unfortunately has one major limitation, its allows to assign only simply types like ints (for example as count query result) or varchar, without possibility to select multiple fields or list of rows.

However, there is simple solution that gets around this problem, by parsing query result to json and reformatting by hibernate converter. 

---

#### Versions
- `kotlin:1.6.21`
- `postgres:14.5`
- `hibernate:5.6.10`

---

### Procedure
```sql
create or replace
function calculate_summary(parent_id bigint) returns text as $$
select
	json_build_object(
	'min_id', min(t.id), 
	'max_id', max(t.id), 
	'elements_count', count(t))::text
from
	test t
where
	t.parent_id = parent_id;

$$ language sql;
```

### Converter

```kotlin
class SummaryConverter : AttributeConverter<Summary, String> {
  override fun convertToDatabaseColumn(attribute: Summary?) = null

  override fun convertToEntityAttribute(jsonText: String?): Summary = 
      jsonText.let { jacksonObjectMapper().readValue(it, Summary::class.java) }
}

data class Summary(
  @JsonProperty("min_id")
  val minId: Long,
  @JsonProperty("max_id")
  val maxID: Long,
  @JsonProperty("elements_count")
  val elementsCount: Long
)
```

### Field in entity

```kotlin
@Formula("(select calculate_summary(id))")
@Convert(converter = SummaryConverter::class)
val summary: Summary
```

This approach allows to collect any result from the database, both single and a list of elements.