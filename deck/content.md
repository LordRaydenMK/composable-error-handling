## Intro to FP in Kotlin

### Stojan Anastasov



### What is Functional Programming

Functional programming is building programs by composing **pure** functions.

Note: Mathematical functions, not CS functions



## Characteristics of Pure Functions

- **Totality** <!-- .element: class="fragment" data-fragment-index="1" -->
- **Determinism** <!-- .element: class="fragment" data-fragment-index="2" -->
- **Purity** <!-- .element: class="fragment" data-fragment-index="3" -->




## Totality

A function must yield a value for every possible input.

Note: Throwing exceptions is not allowed, nulls are not allowed.


### A total function

```kotlin
fun add1(a: Int): Int = a + 1
```


### NOT total function

```kotlin
fun double(a: Int): Int =
    if (a == 5) a * 2
    else throw IllegalArgumentException("Not 5")
```



## Determinism

A function must yield the same value for the same input.

Note: Running the function 1000 times for the same input must return the same output.


### Deterministic

```kotlin
fun add1(a: Int): Int = a + 1
```


### NOT deterministic

```kotlin
fun number(): Int = Random.nextInt()
```



## Purity

A function's only effect must be the computation of its return value.


### PURE

```kotlin
fun add1(a: Int): Int = a + 1
```


### NOT Pure

```kotlin
fun add2(a: Int): Int {
    println("I'm not pure!")
    return a + 2
}
```



## Benefits

You can understand a program by understanding the behaviors of the individual parts. Less stuff to keep in your head.



## Textbook FP

```kotlin
fun fib(list: List<Int>): List<Int> =
    when {
        list.isEmpty() -> list
        else -> {
            val first = list.first()
            val tail = list.drop(1)
            val (before, after) = tail.partition { it < first }
            sort(before) + first + sort(after)
        }
    }
```

Note: A sorted empty list is an empty list  
A sorted non-empty list with a first element **first** is the sorted elements smaller than **first** concatenating it with **first** then concatenating with the sorted elements bigger than **first**.  
I usually use List.sort()



## Breaking Totality

- A function that never completes
- A function that throws Exceptions


## Life Before Exceptions
```
int parse_config(config *cfg) {
  FileHandle handle;
  char *bytes;

  handle = new_handle();

  handle = file_open("config.cfg");

  if (handle == NULL) return -1;

  char *bytes = file_read(handle);

  if (bytes == NULL) return -2;

  ...

  return 0;
}
```

Note: we used sentinel values to indicate failure. You have to read the documentation or even ignore errors.


## Exceptions

Programmers could avoid tangling error-handling concerns with application logic, and benefit from the short-circuiting behavior of exceptions.


## The problem with Exceptions

In a language that supports them, you have no guarantee where they will occur. Which means they can occur everywhere.

Note: This leads to defensive programming, catch log and rethrow etc... Expensive on the JVM, lost in async boundaries.


## Can we have both totality and clean code

```
fun <A> head(list: List<A>): A = list[0]
```

Not total, throws Exception if the list is empty.


## Modeling optionality

```kotlin
sealed class Maybe<out A> {
    class There<A>(val value: A) : Maybe<A>()
    object NotThere : Maybe<Nothing>()
}
```

```
fun <A> head(list: List<A>): Maybe<A> = when {
    list.isEmpty() -> There(list[0])
    else -> NotThere
}
```
<!-- .element: class="fragment" data-fragment-index="1" -->

Note: Now the user is forced to deal with the facts the value could be missing


## Maybe

Unfortunately Maybe tells us nothing about the problem.



## Modeling exceptionality

```
sealed class Result<out E, out A> {
    data class Error<out E>(val error: E) : Result<E, Nothing>()
    data class Success<out A>(val value: A) : Result<Nothing, A>()
}
```


## Parsing config

```
fun fileOpen(path: String): Result<FileError, FileHandle> = TODO()
```

```
fun parseConfig(): Result<FileError, Config> =
    when(val handle = fileOpen("config")) {
        is Result.Error -> Result.Error(handle.error)
        is Result.Success -> {
            when(val read = fileRead(handle.value)) {
                is Result.Error ->  Result.Error(read.error)
                is Result.Success -> TODO()
            }
        }
    }
```
<!-- .element: class="fragment" data-fragment-index="1" -->

Boilerplate when chaining functions... 
<!-- .element: class="fragment" data-fragment-index="2" -->

Note: Fortunately there is a pattern


## Chain

```
fun <E, A, B> Result<E, A>.chain(f: (A) -> Result<E, B>): Result<E, B> =
    when (this) {
        is Result.Error -> this
        is Result.Success -> f(value)
    }
```


## Using Chain

```
fun parseConfig(): Result<FileError, Config> =
        fileOpen("config")
            .chain { fileHandle ->
                fileRead(fileHandle)
                    .chain { bytes ->
                        TODO()
                    }
            }
```

Note: Boilerplate removed!!


## Change

```
fun <B> change(f: (A) -> B): Result<E, B> =
        when (this) {
            is Error -> this
            is Success -> Success(f(value))
        }
```

Note: Updating the wrapped value



## Pyramid of doom

![Pyramid of doom](https://ih0.redbubble.net/image.291385935.1012/ap,550x550,16x12,1,transparent,t.u1.png)




## Using Arrow

```
fun parseConfig(): Either<FileError, Config> =
    binding<FileError, Config> {
        val handle = fileOpen("config").bind()
        val read = fileRead(handle).bind()
        TODO()
    }.fix()
```

Note: Result is Either  
change is map  
chain is flatMap  
Comprehensions  
We can have 20 operations and on the first error it short circuits



## Let's go back in time

x = 10
y = 20
z = x + y <!-- .element: class="fragment" data-fragment-index="1" -->
z = 10 + y <!-- .element: class="fragment" data-fragment-index="2" -->
z = 10 + 20 <!-- .element: class="fragment" data-fragment-index="3" -->
z = 30 <!-- .element: class="fragment" data-fragment-index="4" -->

## Pure functions

## Determinism & Purity

```
class Logging(file: File) {
    private val ostream: OutputStream

    init {
        ostream = FileOutputStream(file)
    }
}
```

This constructor may throw exceptions, which is very unexpected!

Note: This can work fine in production for months/years and then fail because of no disk space.


## Determinism & Purity

Non-determinism makes testing and reasoning about the functions particularly difficult. External state may alter the behavior of the function.



## Is FP practical

If all the functions are  functions total, deterministic, and pure how do we do anything useful?


## How do we

- Print to the console
- Display UI
- Execute network requests
- Update the database


## Console IO