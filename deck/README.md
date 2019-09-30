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

---

## Android Activity Lifecycle

<img src="http://landenlabs.com/android/info/activity-life-cycle/activity-life-cycle-v2.png" alt="Android activity lifecycle" width="400px">

>--

## Solving complex problems

- Split problem into smaller problems
- Solve small problems
- Combine the solutions of the small problems

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

>--

## The Domain

```kotlin

import java.time.LocalDate

data class Email(val email: String)

data class String50(val value: String)

data class User(
    val email: Email,
    val firstName: String50,
    val lastName: String50,
    val dateOfBirth: LocalDate
)
```

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

## Error reporting

<img src="http://www.icge.co.uk/languagesciencesblog/wp-content/uploads/2014/04/you_shall_not_pass1.jpg" alt="You shall not pass">

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

fun validateName(name: String?): Boolean {
    require(!name.isNullOrBlank() && name.length < 50)
    { "Name must be between 1 and 50 chars, found: '$name'" }
    return true
}
```

IllegalArgumentException if the predicate is false

>--

## Return type is always true

```kotlin

fun validateEmailUnit(email: String?): Unit =
    require(email != null && email.contains('@'))
    { "Email must contain @, found: '$email'" }

fun validateNameUnit(name: String?): Unit =
    require(!name.isNullOrBlank() && name.length < 50)
    { "Name must be between 1 and 50 chars, found: '$name'" }

fun validateDateOfBirthUnit(dob: String?): Unit = TODO()

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

We only get the first error!

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

fun validateEmail(email: String?): ErrorMsg? =
    if (email != null && email.contains('@')) null
    else "Email must contain @, found: '$email'"

fun validateName(name: String?): ErrorMsg? =
    if (!name.isNullOrBlank() && name.length < 50) null
    else "Name must be between 1 and 50 chars, found: '$name'"

fun validateDateOfBirth(dob: String): ErrorMsg? = TODO()
```

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

## Errors as Values

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

---

## Can we do better

<img src="https://aaronsoundguy.files.wordpress.com/2014/01/famous-characters-troll-face-challenge-accepted-256559.jpg" alt="Challenge accepted" width=400px>

>--

## ValidationResult

```kotlin

sealed class ValidationResult<out E, out A> {
    data class Valid<A>(val a: A) : ValidationResult<Nothing, A>()
    data class Invalid<E>(val e: E) : ValidationResult<E, Nothing>()
}
```

```kotlin

fun <A> valid(a: A): ValidationResult<Nothing, A> =
    ValidationResult.Valid(a)

fun <E> invalid(e: E): ValidationResult<E, Nothing> =
    ValidationResult.Invalid(e)
```

>--

## ValidationResult in the small

```kotlin

fun validateEmail(email: String?): ValidationResult<String, Email> =
    if (email != null && email.contains('@')) valid(Email(email))
    else invalid("Email must contain @, found: '$email'")

fun validateName(name: String?): ValidationResult<String, String50> =
    if (!name.isNullOrBlank() && name.length < 50) valid(String50(name))
    else invalid("Name must be between 1 and 50 chars, found: '$name'")

fun validateDateOfBirth(dob: String?): ValidationResult<String, LocalDate> = TODO()
```

>--

## Combining ValidationResult

```kotlin

typealias Valid<A> = ValidationResult.Valid<A>
typealias Invalid<E> = ValidationResult.Invalid<E>

fun <E, A, B, C> map(
            combine: (E, E) -> E,
            a: ValidationResult<E, A>,
            b: ValidationResult<E, B>,
            f: (A, B) -> C
        ): ValidationResult<E, C> =
            if (a is Valid && b is Valid) valid(f(a.a, b.a))
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
