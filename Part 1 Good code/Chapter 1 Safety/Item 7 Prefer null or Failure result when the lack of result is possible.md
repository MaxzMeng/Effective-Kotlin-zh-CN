## Item 7: Prefer `null` or `Failure` result when the lack of result is possible

Sometimes, a function cannot produce its desired result. A few common examples are:

- We try to get data from some server, but there is a problem with our internet connection
- We try to get the first element that matches some criteria, but there is no such element in our list
- We try to parse an object from the text, but this text is malformatted

There are two main mechanisms to handle such situations:

- Return a `null` or a sealed class indicating failure (that is often named `Failure`)
- Throw an exception

There is an important difference between those two. Exceptions should not be used as a standard way to pass information. **All exceptions indicate incorrect, special situations and should be treated this way. We should use exceptions only for exceptional conditions**(Effective Java by Joshua Bloch)**.** Main reasons for that are:

- The way exceptions propagate is less readable for most programmers and might be easily missed in the code.
- In Kotlin all exceptions are unchecked. Users are not forced or even encouraged to handle them. They are often not well documented. They are not really visible when we use an API. 
- Because exceptions are designed for exceptional circumstances, there is little incentive for JVM implementers to make them as fast as explicit tests.
- Placing code inside a try-catch block inhibits certain optimizations that the compiler might otherwise perform.

On the other hand, `null` or `Failure` are both perfect to indicate an expected error. They are explicit, efficient, and can be handled in idiomatic ways. This is why the rule is that **we should prefer returning `null` or `Failure` when an error is expected, and throwing an exception when an error is not expected.** Here are some examples:



``` kotlin
inline fun <reified T> String.readObjectOrNull(): T? {
   //...
   if (incorrectSign) {
       return null
   }
   //...
   return result
}

inline fun <reified T> String.readObject(): Result<T> {
   //...
   if (incorrectSign) {
       return Failure(JsonParsingException())
   }
   //...
   return Success(result)
}

sealed class Result<out T>
class Success<out T>(val result: T) : Result<T>()
class Failure(val throwable: Throwable) : Result<Nothing>()

class JsonParsingException : Exception()
```

Errors indicated this way are easier to handle and harder to miss. When we choose to use `null`, clients handling such a value can choose from the variety of null-safety supporting features like a safe-call or the Elvis operator:

``` kotlin
val age = userText.readObjectOrNull<Person>()?.age ?: -1
```

When we choose to return a union type like `Result`, the user will be able to handle it using the when-expression:

``` kotlin
val personResult = userText.readObject<Person>()
val age = when(personResult) {
    is Success -> personResult.value.age
    is Failure -> -1
}
```

Using such error handling is not only more efficient than the try-catch block but often also easier to use and more explicit. An exception can be missed and can stop our whole application. Whereas a `null` value or a sealed result class needs to be explicitly handled, but it won’t interrupt the flow of the application. 

Comparing nullable result and a sealed result class, we should prefer the latter when we need to pass additional information in case of failure, and `null` otherwise. Remember that `Failure` can hold any data you need. 

It is common to have two variants of functions - one expecting that failure can occur and one treating it as an unexpected situation. A good example is that `List` has both:

- `get` which is used when we expect an element to be at the given position, and if it is not there, the function throws `IndexOutOfBoundsException`.
- `getOrNull`, which is used when we suppose that we might ask for an element out of range, and if that happens, we’ll get `null`. 

It also support other options, like `getOrDefault`, that is useful in some cases but in general might be easily replaced with `getOrNull` and Elvis operator `?:`. 

This is a good practice because if developers know they are taking an element safely, they should not be forced to handle a nullable value, and at the same time, if they have any doubts, they should use `getOrNull` and handle the lack of value properly.