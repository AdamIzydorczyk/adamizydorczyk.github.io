---
title: "Flattening of complex objects in Kotlin"
classes: wide
categories:
- programming
tags:
- kotlin
---

Recently I had to map complex object to flat structure without nesting.

To not do to do it manually for every type, I prepared simple recursive solution based on reflection.

#### Implementation
```kotlin
val listOfTypesToFetch = listOf(String::class, Long::class).map { it.createType() }

fun flatMapObjectFields(field: KProperty1<*, *>, instance: Any?): List<FieldWithInstance> =
    if (field.returnType in listOfTypesToFetch) {
        listOf(FieldWithInstance(field, instance))
    } else {
        field.returnType.jvmErasure.declaredMemberProperties.flatMap {
            flatMapObjectFields(it, field.call(instance))
        }
    }

data class FieldWithInstance(val field: KProperty1<*, *>, val instance: Any?)
```

#### Example invocation

```kotlin
val foo = Foo(1, "foo", Bar(2, "bar"))

Foo::class.declaredMemberProperties.flatMap { flatMapObjectFields(it, foo) }
    .groupBy(keySelector = { it.field.toString() }, valueTransform = {it.field.call(it.instance)})
    .mapValues { it.value.first() }
```

#### Result
```json
{
  "val Bar.bar: kotlin.String": "bar",
  "val Bar.id: kotlin.Long": 2,
  "val Foo.foo: kotlin.String": "foo",
  "val Foo.id: kotlin.Long": 1
}
```