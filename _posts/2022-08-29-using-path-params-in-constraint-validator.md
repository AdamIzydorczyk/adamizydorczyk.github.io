---
title: "Using path params in constraint validator"
classes: wide
categories:
- programming
tags:
- kotlin
- spring
---

In many situations I faced with problem when I wanted to use previous state of object during update in implementations of `javax.validation.ConstraintValidator`.
The problematic part is that into validator is passed only value of field or class instance that is annotated by our custom validation.
Fortunately exists workaround for that problem.

Example with validator that preventing of using same id on field as in request path.
#### Versions
- `kotlin:1.6.21`
- `spring-boot-starter-validation:2.7.2`
- `spring-boot-starter-web:2.7.2`
#### Annotation
```kotlin
@MustBeDocumented
@Constraint(validatedBy = [NotSelfAssignedValidator::class])
@Target(allowedTargets = [FIELD])
@Retention(RUNTIME)
annotation class NotSelfAssigned(
  val message: String = IS_SELF_ASSIGNED,
  val groups: Array<KClass<*>> = [OnUpdate::class],
  val payload: Array<KClass<out Payload>> = [])
```

#### Validator

```kotlin
@Component
@RequestScope
class NotSelfAssignedValidator : ConstraintValidator<NotSelfAssigned, Long?> {

  @Autowired
  private lateinit var request: HttpServletRequest

  override fun isValid(changedId: Long?, context: ConstraintValidatorContext?): Boolean {
    return changedId?.let(this::isNotSelfAssigned) ?: true
  }

  @Suppress("UNCHECKED_CAST")
  private fun isNotSelfAssigned(changedId: Long): Boolean {
    val patchVariables =
      request.getAttribute(URI_TEMPLATE_VARIABLES_ATTRIBUTE) as LinkedHashMap<String, String>
    val pathId = patchVariables[PATH_VARIABLE_NAME]?.toLong() ?: -1
    return changedId != pathId
  }

}
```

Obviously that solution can by used for more complex cases, for example witch finding current object state by passed id and using it to perform validation.