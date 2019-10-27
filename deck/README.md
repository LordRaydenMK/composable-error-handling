## Composable Error Handling

### Stojan Anastasov

@s_anastasov

>--

## Functional Error Handling

### Stojan Anastasov

@s_anastasov

---

## Solving problems

As developers we solve complex problems.

>--

## Android Activity Lifecycle

<img src="http://landenlabs.com/android/info/activity-life-cycle/activity-life-cycle-v2.png" alt="Android activity lifecycle" width="400px">

>--

## Solving complex problems

- Split problem into smaller problems <!-- .element: class="fragment" data-fragment-index="1" -->
- Write code solving the small problems <!-- .element: class="fragment" data-fragment-index="2" -->
- Combine the solutions of the small problems <!-- .element: class="fragment" data-fragment-index="3" -->

---

## Data Validation

We have a Sign Up form with:

- Email
- First Name
- Last Name
- Date of Birth

>--

## Validation rules

- Email must contain @
- First Name and Last Name can't be blank. Max length 50.
- Date of Birth must be formatted as YYYY-MM-DD

>--

## The DTO

```kotlin

data class UserDto(
    val email: String?,
    val firstName: String?,
    val lastName: String?,
    val dateOfBirth: String?
)
```

*Postel's law*: Be conservative in what you do, be liberal in what you accept from others.
<!-- .element: class="fragment" data-fragment-index="1" -->

>--

## The Domain

```kotlin

data class Email(val email: String) { companion object }
```
<!-- .element: class="fragment" data-fragment-index="1" -->

```kotlin

data class String50(val value: String) { companion object }
```
<!-- .element: class="fragment" data-fragment-index="2" -->

```kotlin

import java.time.LocalDate

data class User(
    val email: Email,
    val firstName: String50,
    val lastName: String50,
    val dateOfBirth: LocalDate
) { companion object }
```
<!-- .element: class="fragment" data-fragment-index="3" -->

Note: We can easily combine 4 fields into a data class

---

## Validating email

Email must contain @

```kotlin

fun validateEmail(email: String?): Boolean =
    email != null && email.contains('@')
```

Note: Sub-problem 1

>--

## Validating First/Last Name

First Name and Last Name can't be blank. Max length 50 (DB limit).

```kotlin

fun validateName(name: String?): Boolean =
    !name.isNullOrBlank() && name.length < 50
```

Note: Sub-problem 2

>--

## Validating DOB

Date of Birth must be formatted as YYYY-MM-DD

```kotlin

import java.time.LocalDate
import java.time.format.DateTimeParseException

fun validateDateOfBirth(dob: String?): Boolean =
    try {
        LocalDate.parse(dob)
        true
    } catch (e: DateTimeParseException) {
        false
    }
```

Note: Sub-problem 3

>--

## Validating User

```kotlin

import java.time.LocalDate
import java.time.format.DateTimeParseException

fun validateUser(
    email: String?,
    firstName: String?,
    lastName: String?,
    dob: String?
): Boolean = validateEmail(email)
        && validateName(firstName)
        && validateName(lastName)
        && validateDateOfBirth(dob)
```

Note: Combining the solutions

>--

## Usage (A) -> Boolean

```kotlin

validateUser("stojan", null, "", "August")
// false
```

<img src="http://www.icge.co.uk/languagesciencesblog/wp-content/uploads/2014/04/you_shall_not_pass1.jpg" alt="You shall not pass" />
<!-- .element: class="fragment" data-fragment-index="1" -->

>--

## Booleans

- Composes well

- Bad error messages

Boolean -> True | False

---

## Exceptions

```kotlin

fun validateEmailBool(email: String?): Boolean {
    require(email != null && email.contains('@'))
    { "Email must contain @, found: '$email'" }
    return true
}
```
<!-- .element: class="fragment" data-fragment-index="1" -->

```kotlin

fun validateName(name: String?): Boolean {
    require(!name.isNullOrBlank() && name.length < 50)
    { "Name must be between 1 and 50 chars, found: '$name'" }
    return true
}
```
<!-- .element: class="fragment" data-fragment-index="2" -->

IllegalArgumentException if the predicate is false
<!-- .element: class="fragment" data-fragment-index="3" -->

>--

## Return type is always true

```kotlin

fun validateEmailUnit(email: String?): Unit =
    require(email != null && email.contains('@'))
    { "Email must contain @, found: '$email'" }

fun validateNameUnit(name: String?): Unit =
    require(!name.isNullOrBlank() && name.length < 50)
    { "Name must be between 1 and 50 chars, found: '$name'" }

fun validateDateOfBirthUnit(dob: String?): Unit {
    LocalDate.parse(dob)
}
```

>--

## Composing

```kotlin

fun validateUserUnit(
    email: String?,
    firstName: String?,
    lastName: String?,
    dob: String?
): Unit {
    validateEmailUnit(email)
    validateNameUnit(firstName)
    validateNameUnit(lastName)
    validateDateOfBirthUnit(dob)
}
```

>--

## Usage (A) -> Unit + Exception

```kotlin

validateUserUnit("stolea@gmail.com", "Stojan", "An", "1995-10-10")
```

```kotlin

val result = try {
    validateUserUnit("stojan", null, "", "August")
    "Valid"
} catch (e: Exception) {
    e.message!!
}
result
// Email must contain @, found: 'stojan'
```
<!-- .element: class="fragment" data-fragment-index="1" -->

We only get the first error! <!-- .element: class="fragment" data-fragment-index="1" -->

>--

## Accumulate errors

```kotlin

fun validateUserAccumulateErrors(
    email: String?,
    firstName: String?,
    lastName: String?,
    dob: String?
): Unit {
    val errors = mutableSetOf<String>()

    try {
        validateEmailUnit(email)
    } catch (e: IllegalArgumentException) {
        errors.add(e.message!!)
    }

    // TODO: firstName, lastName, dob

    if (errors.isNotEmpty()) {
        throw IllegalArgumentException("Validation errors: ${errors.joinToString()}")
    }
}
```

Glue code to the MAX!

>--

## Problems with Exceptions

- Boilerplate code
- Throwing Exceptions is expensive on JVM
- Dos not fit on a slide

Note: we can fix some of those problems

---

## Errors as values

```kotlin

typealias ErrorMsg = String
```

```kotlin

fun validateEmail(email: String?): ErrorMsg? =
    if (email != null && email.contains('@')) null
    else "Email must contain @, found: '$email'"
```
<!-- .element: class="fragment" data-fragment-index="1" -->

```kotlin

fun validateName(name: String?): ErrorMsg? =
    if (!name.isNullOrBlank() && name.length < 50) null
    else "Name must be between 1 and 50 chars, found: '$name'"

fun validateDateOfBirth(dob: String): ErrorMsg? = TODO()
```
<!-- .element: class="fragment" data-fragment-index="2" -->

>--

## Composing values

```kotlin

fun validateUser(
    email: String?,
    firstName: String?,
    lastName: String?,
    dob: String?
): ErrorMsg? {
    val errorMsg = listOfNotNull(
        validateEmail(email),
        validateName(firstName),
        validateName(lastName),
        validateDateOfBirth(dob)
    ).joinToString()
    return if (errorMsg.isEmpty()) null else errorMsg
}
```

>--

## ErrorRes

- Composable
- Good error messages

- Developer friendly ?

>--

## Error prone

```kotlin

    val email: String? = "stolea@gmail.com"
    val emailErr: ErrorMsg? = validateEmail(email)
    if (emailErr == null) {
        Email(email!!) // <-- Error prone
    }
```

validateEmail already does a null check

>--

## Smart cast

<img src="https://i.stack.imgur.com/kLE8Y.png" alt="Kotlin smart cast">

>--

## Can we do better

<img src="https://aaronsoundguy.files.wordpress.com/2014/01/famous-characters-troll-face-challenge-accepted-256559.jpg" alt="Challenge accepted" width=400px>

---

## ValidationResult

```kotlin

sealed class ValRes<out E, out A> {
    data class Valid<A>(val a: A) : ValRes<Nothing, A>()
    data class Invalid<E>(val e: E) : ValRes<E, Nothing>()
}
```

```kotlin

fun <A> valid(a: A): ValRes<Nothing, A> = ValRes.Valid(a)

fun <E> invalid(e: E): ValRes<E, Nothing> = ValRes.Invalid(e)
```
<!-- .element: class="fragment" data-fragment-index="1" -->

>--

## ValidationResult in the small

```kotlin

fun validateEmail(email: String?): ValRes<String, Email> =
    if (email != null && email.contains('@')) valid(Email(email))
    else invalid("Email must contain @, found: '$email'")
```

```kotlin

fun validateName(name: String?): ValRes<String, String50> =
    if (!name.isNullOrBlank() && name.length < 50) valid(String50(name))
    else invalid("Name must be between 1 and 50 chars, found: '$name'")
```
<!-- .element: class="fragment" data-fragment-index="1" -->

```kotlin

fun validateDateOfBirth(dob: String?): ValRes<String, LocalDate> = TODO()
```
<!-- .element: class="fragment" data-fragment-index="1" -->

>--

## Combining ValRes

```kotlin

typealias Valid<A> = ValRes.Valid<A>
typealias Invalid<E> = ValRes.Invalid<E>

fun <E, A, B> tupled(
            combine: (E, E) -> E,
            a: ValRes<E, A>,
            b: ValRes<E, B>
        ): ValRes<E, Pair<A, B>> =
            if (a is Valid && b is Valid) valid(Pair(a.a, b.a))
            else if (a is Invalid && b is Invalid) invalid(combine(a.e, b.e))
            else if (a is Invalid) invalid(a.e)
            else if (b is Invalid) invalid(b.e)
            else throw IllegalStateException("This is impossible")
```

>--

## Usage

```kotlin

validateEmail("stolea@gmail.com")
// Valid(a=Email(email=stolea@gmail.com))
```

```kotlin

validateEmail("email")
// Invalid(e=Email must contain @, found: 'email')
```
<!-- .element: class="fragment" data-fragment-index="1" -->

```kotlin

tupled(
    {e1, e2 -> "$e1, $e2"},
    validateEmail("stojan"),
    validateName(null)
)
// Invalid(e=Email must contain @, found: 'stojan', Name must be between 1 and 50 chars, found: 'null')
```
<!-- .element: class="fragment" data-fragment-index="1" -->

>--

```kotlin

data class Triple<A, B, C>(val a: A, val b: B, val c: C)

fun <E, A, B, C> tupled(
            combine: (E, E, E) -> E,
            a: ValRes<E, A>,
            b: ValRes<E, B>,
            c: ValRes<E, C>
        ): ValRes<E, Triple<A, B, C>> = TODO()
```

---

## Arrow-kt

<img src="images/arrow.svg" alt="arrow logo">

Functional companion to Kotlin's Standard Library

arrow-kt.io

>--

## Validated

```kotlin
sealed class Validated<out E, out A> {
    data class Valid<out A>(val a: A) : Validated<Nothing, A>()
    data class Invalid<out E>(val e: E) : Validated<E, Nothing>()
}
```

```kotlin

import arrow.core.*

typealias ValidatedNel<E, A> = Validated<Nel<E>, A>
```
<!-- .element: class="fragment" data-fragment-index="1" -->

Note: Nel is short for NonEmptyList

>--

## ValidationResult

```kotlin

typealias ValidationResult<A> = ValidatedNel<String, A>
```

```kotlin

fun Email.Companion.create(email: String?): ValidationResult<Email> =
    if (email != null && email.contains('@')) Email(email).valid()
    else "Email must contain @, found: '$email'".invalidNel()
```

>--

```kotlin

fun String50.Companion.create(name: String?): ValidationResult<String50> =
    if (!name.isNullOrBlank() && name.length < 50) String50(name).valid()
    else "Name must be between 1 and 50 chars, found: '$name'".invalidNel()
```

```kotlin

fun validateDateOfBirth(dob: String?): ValidationResult<LocalDate> =
    try {
        LocalDate.parse(dob).valid()
    } catch (e: DateTimeParseException) {
        "Date of Birth must be a valid date, found: '$dob'".invalidNel()
    }
```
<!-- .element: class="fragment" data-fragment-index="1" -->

>--

```kotlin

import arrow.core.extensions.nonemptylist.semigroup.semigroup
import arrow.core.extensions.validated.applicative.applicative

fun User.Companion.create(
    email: String?,
    firstName: String?,
    lastName: String?,
    dob: String?
): ValidationResult<User> =
    ValidationResult.applicative(Nel.semigroup<String>())
        .tupled(
            Email.create(email),
            String50.create(firstName),
            String50.create(lastName),
            validateDateOfBirth(dob)
        )
        .fix()
        .map { User(it.a, it.b, it.c, it.d) }
```

>--

```kotlin

User.create(
    email = "stolea@gmail.com",
    firstName = "Stojan",
    lastName = "Anastasov",
    dob = "1991-10-10"
)
// Valid(a=User(email=Email(email=stolea@gmail.com), firstName=String50(value=Stojan), lastName=String50(value=Anastasov), dateOfBirth=1991-10-10))
```

>--

```kotlin

User.create(
    email = "",
    firstName = "   ",
    lastName = "a",
    dob = "10.10.1992"
)
// Invalid(e=NonEmptyList(all=[Date of Birth must be a valid date, found: '10.10.1992', Email must contain @, found: '', Name must be between 1 and 50 chars, found: '   ']))
```

>--

## Learn more

arrow-kt.io

Patterns -> Error Handling

---

## Composition

How do things compose

>--

## Lego block

<img src="images/lego.jpg" alt="Lego block" />

>--

## Lego combined

<img src="images/lego-combined.jpeg" alt="Lego combined">

>--

## Lego Falcon

<img src="images/lego-falcon.jpg" alt="Milenium Falcon Lego">

>--

## Combining functions

```kotlin

// (String?) -> ValidationResult<Email>
fun validateEmail(email: String?): ValidationResult<Email> = TODO()
```

```kotlin

// (String?, String?, String?, String?) -> ValidationResult<User>
fun validateUser(
    email: String?,
    firstName: String?,
    lastName: String?,
    dob: String?): ValidationResult<User> = TODO()
```
<!-- .element: class="fragment" data-fragment-index="1" -->

---

## Summary

We can validate data using different technics

- (A) -> Boolean
- (A) -> Unit + Exceptions <!-- .element: class="fragment" data-fragment-index="1" -->
- (A) -> ErrorMsg? <!-- .element: class="fragment" data-fragment-index="2" -->
- (A) -> ValRes <!-- .element: class="fragment" data-fragment-index="3" -->
- (A) -> Validated (arrow-kt) <!-- .element: class="fragment" data-fragment-index="4" -->

---

## Thank You

### Questions
