---
title: "JOOQ integration with GraphQL"
classes: wide
categories:
- programming
tags:
- postgresql
- jooq
- sql
- kotlin
- spring
- graphql
---

As part of new project, I decided to rid of typical approach to building api based on rest and encouraged by the vision of a solution that is a cure for overfetching, try out graphql-based api.
Looking for various sample implementations on the internet, I quickly realized that most examples are based on static queries, which only stage of parsing the result to json reducing amount of fields to requested by client.
I decided that it's a waste of such solution potential and started looking for the best way to dynamically build queries to reduce set of data on during collecting.

Initially I discovered a already exists library https://github.com/introproventures/graphql-jpa-query, which allowed to map the JPA model directly to graphql's api. Unfortunately, this carried some limitations. Primarily coused bu behavior of hibernate itself (mainly couse one to one and many to one lazy relationships behaviurs). Besides, jpa based solutions reduce access to the native functions of the database.

So I came up with another solution, based on the JOOQ library, to use it to map individual fields from a graphql query to a mapped database model.

---

#### Versions
- `postgres:14.5`
- `jooq:7.1.1`
- `kotlin:1.6.21`
- `spring-boot-starter-graphql:2.7.2`
- `spring-boot-starter-jooq:2.7.2`
- `graphql-java-extended-scalars:18.1`

---

### Graphql shema

```graphql
input NumericCriteria{
    EQ: Int
}

input StringCriteria{
    EQ: String
    LIKE: String
    _LIKE: String
}

input JsonCriteria{
    BY_JSON: JsonNode
}

input Page {
    start: Int
    limit: Int
}

input KashubianEntryCriteriaExpression{
    id: NumericCriteria
    note: StringCriteria
    bases: JsonCriteria
    meanings: MeaningsCriteriaExpression
}
    
input MeaningsCriteriaExpression{
    id: NumericCriteria
    definition: StringCriteria
}

type KashubianEntriesPaged{
    pages: Int
    total: Int
    select: [KashubianEntry]
}

type KashubianEntry {
    id(orderBy: OrderBy): Int
    note(orderBy: OrderBy): String
    bases: JSON
    meanings: [Meaning]
}

type Meaning{
    id(orderBy: OrderBy): Int
    definition(orderBy: OrderBy): String
}
```

As you see in query shema was delivered, pagination, sorting and filtering by each field.

### Implementation

#### Mappings

```kotlin
internal val FIND_ALL_FIELD_TO_JOIN_RELATIONS = mapOf(
  "$KASHUBIAN_ENTRIES_PAGED_TYPE_PREFIX$SELECT_PREFIX$KASHUBIAN_ENTRY_TYPE_PREFIX$MEANINGS_NODE" to
    Triple(meaningTable(),
      entryTable().ID.eq(meaningTable().KASHUBIAN_ENTRY_ID),
      meaningId()))

internal val FIND_ALL_FIELD_TO_COLUMN_RELATIONS = mapOf(
  "$KASHUBIAN_ENTRIES_PAGED_TYPE_PREFIX$SELECT_PREFIX$KASHUBIAN_ENTRY_TYPE_PREFIX$ID_FIELD" to
    entryId(),
  "$KASHUBIAN_ENTRIES_PAGED_TYPE_PREFIX$SELECT_PREFIX$KASHUBIAN_ENTRY_TYPE_PREFIX$NOTE_FIELD" to
    entryNote(),
  "$KASHUBIAN_ENTRIES_PAGED_TYPE_PREFIX$SELECT_PREFIX$KASHUBIAN_ENTRY_TYPE_PREFIX$BASES_FIELD" to
    entryBasesWithAlias(),
  "$KASHUBIAN_ENTRIES_PAGED_TYPE_PREFIX$SELECT_PREFIX$KASHUBIAN_ENTRY_TYPE_PREFIX$MEANINGS_NODE$MEANING_TYPE_PREFIX$ID_FIELD" to
    meaningId(),
  "$KASHUBIAN_ENTRIES_PAGED_TYPE_PREFIX$SELECT_PREFIX$KASHUBIAN_ENTRY_TYPE_PREFIX$MEANINGS_NODE$MEANING_TYPE_PREFIX$DEFINITION_FIELD" to
    meaningDefinition()
)

val CRITERIA_TO_COLUMN_RELATIONS_WITH_JOIN: Map<String, Pair<QueryPart, List<JoinTableWithCondition>>> =
  listOf(
    "$SELECT_PREFIX$ID_FIELD" to (entryTable().ID joinedBy emptyList()),
    "$SELECT_PREFIX$NOTE_FIELD" to (entryTable().NOTE joinedBy emptyList()),
    "$SELECT_PREFIX$BASES_FIELD" to (entryBases() joinedBy emptyList()),
    "$SELECT_PREFIX$MEANINGS_NODE$ID_FIELD" to (meaningTable().ID joinedBy
      listOf(meaningTable() on entryTable().ID.eq(meaningTable().KASHUBIAN_ENTRY_ID))),
    "$SELECT_PREFIX$MEANINGS_NODE$DEFINITION_FIELD" to (meaningTable().DEFINITION joinedBy
      listOf(meaningTable() on entryTable().ID.eq(meaningTable().KASHUBIAN_ENTRY_ID)))
  ).map { criteriaAndField ->
    listOf(".EQ", "._LIKE", ".LIKE", ".BY_JSON").map {
      criteriaAndField.fieldPath() + it to
        (criteriaAndField.fieldWithJoins().field to criteriaAndField.fieldWithJoins().joins)
    }
  }.flatten().associate { it.first to it.second }

const val KASHUBIAN_ENTRIES_PAGED_TYPE_PREFIX = "KashubianEntriesPaged."
const val SELECT_PREFIX = "select"
const val KASHUBIAN_ENTRY_TYPE_PREFIX = "/KashubianEntry"
const val MEANINGS_NODE = ".meanings"
const val MEANING_TYPE_PREFIX = "/Meaning"

const val ID_FIELD = ".id"
const val NOTE_FIELD = ".note"
const val BASES_FIELD = "${BASE_NODE}s"
const val DEFINITION_FIELD = ".definition"

fun entryTable() = Tables.KASHUBIAN_ENTRY.`as`("entry")
fun meaningTable() = Tables.MEANING.`as`("meaning")

fun entryId() = Tables.KASHUBIAN_ENTRY.`as`("entry").ID.`as`("entry_id")
fun entryNote() = entryTable().NOTE.`as`("entry_note")
fun entryBasesWithAlias() = field(select(Routines.findBases(entryTable().ID))).`as`("entry_bases")
fun meaningId() = meaningTable().ID.`as`("meaning_id")
fun meaningDefinition() = meaningTable().DEFINITION.`as`("meaning_definition")
```

In this way the individual field names were linked to their reference in the sql query, it's worth noting that this approach also allowed to very easily add a procedure under individual fields as you can see in the example of the `entryBasesWithAlias()` method.
There are specified 3 kinds of linkings to mappings
- graphql query fields with joins, which allows you to choose which tables to select in order not to join unnecessary ones
- graphql fields to fields, so that only the selected fields are selected in the select clause
- fields with query parameters to fields in the database for filtering

#### Example finder

```kotlin
@QueryMapping
fun findAllKashubianEntries(
  @Argument("page") page: PageCriteria?,
  @Argument("where") where: KashubianEntryCriteriaExpression?,
  env: DataFetchingEnvironment): KashubianEntriesPaged {
  val selectedFields = env.selectionSet.fields
  logger.info("Entries searching by criteria: $where with page: $page and fields: $selectedFields")
  return allKashubianEntriesFinder.findAllKashubianEntries(where, selectedFields, page)
}

@Component
class AllKashubianEntriesFinder(override val dsl: DSLContext) :
    AllFinderBase<KashubianEntryGraphQL>(dsl, KashubianEntryGraphQLMapper()) {
    fun findAllKashubianEntries(where: KashubianEntryCriteriaExpression?,
        selectedFields: MutableList<SelectedField>,
        page: PageCriteria?): KashubianEntriesPaged =
        findAll(KashubianEntryCriteriaExpression::class, where, selectedFields, page)
            .let(::KashubianEntriesPaged)

    override fun idFieldWithAlias() = entryId()

    override fun fieldToJoinRelations() = FIND_ALL_FIELD_TO_JOIN_RELATIONS

    override fun idField(): Field<Long> =
        table().ID

    override fun table() = entryTable()

    override fun fieldToColumnRelations() = FIND_ALL_FIELD_TO_COLUMN_RELATIONS

    override fun relationsWithJoin(): Map<String, Pair<QueryPart, List<JoinTableWithCondition>>> =
        CRITERIA_TO_COLUMN_RELATIONS_WITH_JOIN

    override fun pageTypeName() = KashubianEntriesPaged::class.simpleName!!
}
```

All the implementation details are hidden in the `AllFinderBase` class, for the implementation it is enough to provide mappings for the fields in prepared template method pattern.


#### Base finder

```kotlin
protected fun findAll(criteriaExpressionClass: KClass<out CriteriaExpression>,
  where: CriteriaExpression?,
  selectedFields: MutableList<SelectedField>,
  page: PageCriteria?): GraphQLPagedModel<GraphQLModel> {
  val wheresWithJoins = prepareWheresWithJoins(
    where = where,
    declaredMemberProperties = criteriaExpressionClass.declaredMemberProperties,
    criteriaToColumnRelationsWithJoin = relationsWithJoin()
  )

  val wheres = wheresWithJoins.map { it.condition() }
  val whereJoins = wheresWithJoins.map { it.joins() }
    .flatten()
    .distinct()

  val selectedColumns: MutableSet<SelectFieldOrAsterisk?> = selectColumns(selectedFields,
    fieldToColumnRelations())
  selectedColumns.add(idFieldWithAlias())

  val selectedJoins = selectedFields
    .mapNotNull { fieldToJoinRelations()[it.fullyQualifiedName] }

  selectedJoins.forEach {
    selectedColumns.add(it.idColumn())
  }

  val ordersBy = orderByColumns(selectedFields,
    fieldToColumnRelations())

  val elementsCount = countEntriesIfPaginationFieldsExists(selectedFields, whereJoins, wheres)

  val limit = page?.limit ?: 100
  val start = page?.start ?: 0
  val pageStart = start * limit
  val pageCount: Int = (elementsCount + limit - 1) / limit

  return selectElements(selectedFields,
    selectedColumns,
    selectedJoins,
    whereJoins,
    wheres,
    pageStart,
    limit,
    ordersBy)
    ?.let { GraphQLPagedModel(pageCount, elementsCount, mapper.map(it.fetch())) }
    ?: GraphQLPagedModel(pageCount, elementsCount, emptyList())
}
```

This is the main method responsible for converting graphql to sql queries.

Let's take a look at the individual elements.

##### Base finder

```kotlin
private fun prepareWheresWithJoins(prefix: String = SELECT_PREFIX, where: CriteriaExpression?,
  declaredMemberProperties: Collection<KProperty1<out CriteriaExpression, *>>,
  criteriaToColumnRelationsWithJoin: Map<String, Pair<QueryPart, List<JoinTableWithCondition>>>
): List<Pair<Condition?, List<JoinTableWithCondition>>> = declaredMemberProperties.flatMap {
  flatMapObjectFields(it, where, prefix)
}.filter { it.instance != null }
  .map { Pair(it.field.call(it.instance), it.fieldPath) }
  .filter { it.first != null }
  .map { instanceWithField ->
    val instance = instanceWithField.first!!
    val fieldPath = instanceWithField.second
    val field =
      criteriaToColumnRelationsWithJoin[fieldPath]!!.first as Field<Any>
    val condition = prepareCondition(fieldPath, field, instance)

    val joins = criteriaToColumnRelationsWithJoin[fieldPath]!!.second

    Pair(condition, joins)
  }

private val listOfTypesToFetch =
  listOf(String::class.createType(nullable = true),
    Long::class.createType(nullable = true),
    JsonNode::class.createType(nullable = true))

private fun flatMapObjectFields(field: KProperty1<*, *>,
  instance: Any?,
  fieldPath: String): List<FieldWithInstance> =
  if (field.returnType in listOfTypesToFetch) {
    listOf(FieldWithInstance(field, instance, "$fieldPath.${field.name}"))
  } else {
    field.returnType.jvmErasure.declaredMemberProperties.flatMap {
      instance?.let { inst -> flatMapObjectFields(it, field.call(inst), "$fieldPath.${field.name}") }
        ?: emptyList()
    }
  }

private fun prepareCondition(fieldPath: String,
  field: Field<Any>,
  instance: Any) = when {
  fieldPath.endsWith(".EQ") -> field.eq(instance)
  fieldPath.endsWith("._LIKE") -> field.likeIgnoreCase("%$instance%")
  fieldPath.endsWith(".LIKE") -> field.like("%$instance%")
  fieldPath.endsWith(".BY_JSON") -> jsonContains(field, "$instance")
  else -> null
}

private fun jsonContains(field: Field<Any>, value: String): Condition {
  return condition("{0} @> {1}", field, `val`(value, field))
}
```

These methods are responsible, for preparing where conditions and associated joins.
First, all the fields with criteria are mapped into a flat list of paths and values to be filtered, and then they are linked to the condition based on the field mappings.

```kotlin
protected fun selectColumns(selectedFields: List<SelectedField>,
  fieldToColumnRelations: Map<String, Field<*>>): MutableSet<SelectFieldOrAsterisk?> =
  selectedFields
    .mapNotNull { fieldToColumnRelations[it.fullyQualifiedName] }
    .toMutableSet()
```

The selection of fields for select clause is less complicated, they are taken directly from mappings.

```kotlin
private fun orderByColumns(selectedFields: List<SelectedField>,
  fieldToColumnRelations: Map<String, Field<*>>): List<SortField<*>> =
  selectedFields.filter { it.arguments.isNotEmpty() }.mapNotNull {
    when (it.arguments["orderBy"]) {
      "ASC" -> fieldToColumnRelations[it.fullyQualifiedName]?.asc()
      else -> fieldToColumnRelations[it.fullyQualifiedName]?.desc()
    }
  }
```

Sorting is based on field arguments from the graphql query.

```kotlin
private fun countEntriesIfPaginationFieldsExists(
  selectedFields: MutableList<SelectedField>,
  whereJoins: List<JoinTableWithCondition>,
  wheres: List<Condition?>) =
  when (isContainsPaginationFields(selectedFields)) {
    true -> dsl.select(count())
      .from(table())
      .apply {
        whereJoins.forEach {
          leftJoin(it.joinTable).on(it.joinCondition)
        }

        wheres.forEach {
          where(it)
        }
        logger.info("Count query: $sql")
      }.fetchOne(0, Int::class.java) ?: 0

    false -> 0
  }

private fun isContainsPaginationFields(fields: List<SelectedField>) =
  fields.any {
    it.fullyQualifiedName == "${pageTypeName()}.pages"
      || it.fullyQualifiedName == "${pageTypeName()}.total"
  }
```

If the `total` or `pages` fields are selected, an additional query is generated retrieving the amount of items.

```kotlin
private fun selectElements(selectedFields: MutableList<SelectedField>,
  selectedColumns: MutableSet<SelectFieldOrAsterisk?>,
  selectedJoins: List<Triple<TableImpl<out UpdatableRecordImpl<*>>, Condition, Field<Long>>>,
  whereJoins: List<JoinTableWithCondition>,
  wheres: List<Condition?>,
  pageStart: Int,
  limit: Int,
  ordersBy: List<SortField<*>>): SelectSeekStepN<Record>? {
  return when (isContainsSelectField(selectedFields)) {
    true -> dsl.select(selectedColumns)
      .from(table())
      .apply {
        selectedJoins.forEach {
          leftJoin(it.joinTable()).on(it.joinCondition())
        }
      }
      .where(idField().`in`(
        select(field(idFieldWithAlias().name, Long::class.java))
          .from(select(selectedColumns)
            .from(table())
            .apply {
              whereJoins.forEach {
                leftJoin(it.joinTable).on(it.joinCondition)
              }
              wheres.forEach {
                where(it)
              }
            }
            .orderBy(ordersBy)
            .offset(pageStart)
            .limit(limit).asTable(table()))
      )).orderBy(ordersBy)
      .apply { logger.info("Select query: $sql") }

    false -> null
  }
}
```

And finally part, where the query is built based on the previously extracted fields and conditions.
It is composed of 2 subqueries, one that retrieves ids based on the conditions, and the other retrieves the individual fields selected in the graphql query, this allows you to search fields that aren't in the select clause in the query undependly.

### Example query

```graphql
{
  findAllKashubianEntries(
    page: {start: 0, limit: 2}, where: {note: {_LIKE : "pan"}}
  ) {
    total
    pages
    select {
      id(orderBy: ASC)
      note
      meanings{
        id
        definition
      }
    }
  }
}
```

```json
{
  "data": {
    "findAllKashubianEntries": {
      "total": 13,
      "pages": 7,
      "select": [
        {
          "id": 17,
          "note": "Pan17",
          "meanings": [
            {
              "id": 24,
              "definition": "Kasprzak17"
            },
            {
              "id": 25,
              "definition": "Czechy17"
            }
          ]
        },
        {
          "id": 43,
          "note": "Pani43",
          "meanings": [
            {
              "id": 61,
              "definition": "Lubuskie43"
            }
          ]
        }
      ]
    }
  }
}
```

