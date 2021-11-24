## Item 5: Specify your expectations on arguments and state

When you have expectations, declare them as soon as possible. We do that in Kotlin mainly using:

- `require` block - a universal way to specify expectations on arguments.
- `check` block - a universal way to specify expectations on the state.
- `assert` block - a universal way to check if something is true. Such checks on the JVM will be evaluated only on the testing mode.
- Elvis operator with `return` or `throw`.

Here is an example using those mechanisms:

``` kotlin
// Part of Stack<T>
fun pop(num: Int = 1): List<T> {
   require(num <= size) { 
       "Cannot remove more elements than current size"
   }
   check(isOpen) { "Cannot pop from closed stack" }
   val ret = collection.take(num)
   collection = collection.drop(num)
   assert(ret.size == num)
   return ret
}
```

Specifying experiences this way does not free us from the necessity to specify those expectations in the documentation, but it is really helpful anyway. Such declarative checks have many advantages:

- Expectations are visible even to those programmers who are not reading the documentation.
- If they are not satisfied, the function throws an exception instead of leading to unexpected behavior. It is important that these exceptions are thrown before the state is modified and so we will not have a situation where only some modifications are applied and others are not. Such situations are dangerous and hard to manage[4](chap65.xhtml#fn-pokemon). Thanks to assertive checks, errors are harder to miss and our state is more stable.
- Code is to some degree self-checking. There is less of a need to be unit-tested when these conditions are checked in the code.
- All checks listed above work with smart-casting, so thanks to them there is less casting required.

Let’s talk about different kinds of checks, and why we need them. Starting from the most popular one: the arguments check.

### Arguments

When you define a function with arguments, it is not uncommon to have some expectations on those arguments that cannot be expressed using the type system. Just take a look at a few examples:

- When you calculate the factorial of a number, you might require this number to be a positive integer.
- When you look for clusters, you might require a list of points to not be empty.
- When you send an email to a user you might require that user to have an email, and this value to be a correct email address (assuming that user should check email correctness before using this function).

The most universal and direct Kotlin way to state those requirements is using the `require` function that checks this requirement and throws an exception if it is not satisfied:

``` kotlin
fun factorial(n: Int): Long {
   require(n >= 0)
   return if (n <= 1) 1 else factorial(n - 1) * n
}

fun findClusters(points: List<Point>): List<Cluster> {
   require(points.isNotEmpty())
   //...
}

fun sendEmail(user: User, message: String) {
   requireNotNull(user.email)
   require(isValidEmail(user.email))
   //...
}
```

Notice that these requirements are highly visible thanks to the fact they are declared at the very beginning of the functions. This makes them clear for the user reading those functions (though the requirements should be stated in documentation as well because not everyone reads function bodies).

Those expectations cannot be ignored, because the `require` function throws an
`IllegalArgumentException` when the predicate is not satisfied. When such a block is placed at the beginning of the function we know that if an argument is incorrect, the function will stop immediately and the user won’t miss it. The exception will be clear in opposition to the potentially strange result that might propagate far until it fails. In other words, when we properly specify our expectations on arguments at the beginning of the function, we can then assume that those expectations will be satisfied.

We can also specify a lazy message for this exception in a lambda expression after the call:

``` kotlin
fun factorial(n: Int): Long {
   require(n >= 0) { "Cannot calculate factorial of $n " +
"because it is smaller than 0" }
   return if (n <= 1) 1 else factorial(n - 1) * n
}
```

The `require` function is used when we specify requirements on arguments.

Another common case is when we have expectations on the current state, and in such a case, we can use the `check` function instead.

### State

It is not uncommon that we only allow using some functions in concrete conditions. A few common examples:

- Some functions might need an object to be initialized first.
- Actions might be allowed only if the user is logged in.
- Functions might require an object to be open.

The standard way to check if those expectations on the state are satisfied is to use the `check` function:

``` kotlin
fun speak(text: String) {
   check(isInitialized)
   //...
}

fun getUserInfo(): UserInfo {
   checkNotNull(token)
   //...
}

fun next(): T {
   check(isOpen)
   //...
}
```

The `check` function works similarly to `require`, but it throws an `IllegalStateException` when the stated expectation is not met. It checks if a state is correct. The exception message can be customized using a lazy message, just like with `require`. When the expectation is on the whole function, we place it at the beginning, generally after the `require` blocks. Although some state expectations are local, and `check` can be used later.

We use such checks especially when we suspect that a user might break our contract and call the function when it should not be called. Instead of trusting that they won’t do that, it is better to check and throw an appropriate exception. We might also use it when we do not trust that our own implementation handles the state correctly. Although for such cases, when we are checking mainly for the sake of testing our own implementation, we have another function called `assert`.

### Assertions

There are things we know to be true when a function is implemented correctly. For instance, when a function is asked to return 10 elements we might expect that it will return 10 elements. This is something we expect to be true, but it doesn’t mean we are always right. We all make mistakes. Maybe we made a mistake in the implementation. Maybe someone changed a function we used and our function does not work properly anymore. Maybe our function does not work correctly anymore because it was refactored. For all those problems the most universal solution is that we should write unit tests that check if our expectations match reality:

``` kotlin
class StackTest {
  
   @Test
   fun `Stack pops correct number of elements`() {
       val stack = Stack(20) { it }
       val ret = stack.pop(10)
       assertEquals(10, ret.size)
   }
  
   //...
}
```

Unit tests should be our most basic way to check implementation correctness but notice here that the fact that popped list size matches the desired one is rather universal to this function. It would be useful to add such a check in nearly every `pop` call. Having only a single check for this single use is rather naive because there might be some edge-cases. A better solution is to include the assertion in the function:

``` kotlin
fun pop(num: Int = 1): List<T> {
   //...
   assert(ret.size == num)
   return ret
}
```

Such conditions are currently enabled only in Kotlin/JVM, and they are not checked unless they are enabled using the `-ea` JVM option. We should rather treat them as part of unit tests - they check if our code works as expected. By default, they are not throwing any errors in production. They are enabled by default only when we run tests. This is generally desired because if we made an error, we might rather hope that the user won’t notice. If this is a serious error that is probable and might have significant consequences, use `check` instead. The main advantages of having `assert` checks in functions instead of in unit tests are:

- Assertions make code self-checking and lead to more effective testing.
- Expectations are checked for every real use-case instead of for concrete cases.
- We can use them to check something at the exact point of execution.
- We make code fail early, closer to the actual problem. Thanks to that we can also easily find where and when the unexpected behavior started.

Just remember that for them to be used, we still need to write unit tests. In a standard application run, `assert` will not throw any exceptions.

Such assertions are a common practice in Python. Not so much in Java. In Kotlin feel free to use them to make your code more reliable.

### Nullability and smart casting

Both `require` and `check` have Kotlin contracts that state that when the function returns, its predicate is true after this check.

``` kotlin
public inline fun require(value: Boolean): Unit {
   contract {
       returns() implies value
   }
   require(value) { "Failed requirement." }
}
```

Everything that is checked in those blocks will be treated as true later in the same function. This works well with smart casting because once we check if something is true, the compiler will treat it so. In the below example we require a person’s outfit to be a `Dress`. After that, assuming that the outfit property is final, it will be smart cast to `Dress`.

``` kotlin
fun changeDress(person: Person) {
   require(person.outfit is Dress)
   val dress: Dress = person.outfit
   //...
}
```

This characteristic is especially useful when we check if something is null:

``` kotlin
class Person(val email: String?)

fun sendEmail(person: Person, message: String) {
   require(person.email != null)
   val email: String = person.email
   //...
}
```

For such cases, we even have special functions: `requireNotNull` and `checkNotNull`. They both have the capability to smart cast variables, and they can also be used as expressions to “unpack” it:

``` kotlin
class Person(val email: String?)
fun validateEmail(email: String) { /*...*/ }

fun sendEmail(person: Person, text: String) {
   val email = requireNotNull(person.email)
   validateEmail(email)
   //...
}

fun sendEmail(person: Person, text: String) {
   requireNotNull(person.email)
   validateEmail(person.email)
   //...
}
```

For nullability, it is also popular to use the Elvis operator with `throw` or `return` on the right side. Such a structure is also highly readable and at the same time, it gives us more flexibility in deciding what behavior we want to achieve. First of all, we can easily stop a function using `return` instead of throwing an error:

``` kotlin
fun sendEmail(person: Person, text: String) {
   val email: String = person.email ?: return
   //...
}
```

If we need to make more than one action if a property is incorrectly `null`, we can always add them by wrapping `return` or `throw` into the `run` function. Such a capability might be useful if we would need to log why the function was stopped:

``` kotlin
fun sendEmail(person: Person, text: String) {
   val email: String = person.email ?: run {
       log("Email not sent, no email address")
       return
   }
   //...
}
```

The Elvis operator with `return` or `throw` is a popular and idiomatic way to specify what should happen in case of variable nullability and we should not hesitate to use it. Again, if it is possible, keep such checks at the beginning of the function to make them visible and clear.

### Summary

Specify your expectations to:

- Make them more visible.
- Protect your application stability.
- Protect your code correctness.
- Smart cast variables.

Four main mechanisms we use for that are:

- `require` block - a universal way to specify expectations on arguments.
- `check` block - a universal way to specify expectations on the state.
- `assert` block - a universal way to test in testing mode if something is true.
- Elvis operator with `return` or `throw`.

You might also use `throw` to throw a different error.