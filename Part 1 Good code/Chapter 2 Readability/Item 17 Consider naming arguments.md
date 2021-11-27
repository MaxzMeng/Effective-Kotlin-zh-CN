## Item 17: Consider naming arguments

When you read a code, it is not always clear what an argument means. Take a look at the following example:

``` kotlin
val text = (1..10).joinToString("|")
```

What is `"|"`? If you know `joinToString` well, you know that it is the `separator`. Although it could just as well be the `prefix`. It is not clear at all[3](chap65.xhtml#fn-footnote_21_note). We can make it easier to read by clarifying those arguments whose values do not clearly indicate what they mean. The best way to do that is by using named arguments:

``` kotlin
val text = (1..10).joinToString(separator = "|")
```

We could achieve a similar result by naming variable:

``` kotlin
val separator = "|"
val text = (1..10).joinToString(separator)
```

Although naming the argument is more reliable. A variable name specifies developer intention, but not necessarily correctness. What if a developer made a mistake and placed the variable in the wrong position? What if the order of parameters changed? Named arguments protect us from such situations while named values do not. This is why it is still reasonable to use named arguments when we have values named anyway:

``` kotlin
val separator = "|"
val text = (1..10).joinToString(separator = separator)
```

### When should we use named arguments?

Clearly, named arguments are longer, but they have two important advantages:

- Name that indicates what value is expected. 
- They are safer because they are independent of order.

The argument name is important information not only for a developer using this function but also for the one reading how it was used. Take a look at this call:

``` kotlin
sleep(100)
```

How much will it sleep? 100 ms? Maybe 100 seconds? We can clarify it using a named argument:

``` kotlin
sleep(timeMillis = 100)
```

This is not the only option for clarification in this case. In statically typed languages like Kotlin, the first mechanism that protects us when we pass arguments is the parameter type. We could use it here to express information about time unit:

``` kotlin
sleep(Millis(100))
```

Or we could use an extension property to create a DSL-like syntax:

``` kotlin
sleep(100.ms)
```

Types are a good way to pass such information. If you are concerned about efficiency, use inline classes as described in *Item 46: Use inline modifier for functions with parameters of functional types*. They help us with parameter safety, but they do not solve all problems. Some arguments might still be unclear. Some arguments might still be placed on wrong positions. This is why I still suggest considering named arguments, especially for parameters:

- with default arguments,
- with the same type as other parameters,
- of functional type, if they’re not the last parameter.

### Parameters with default arguments

When a property has a default argument, we should nearly always use it by name. Such optional parameters are changed more often than those that are required. We don’t want to miss such a change. Function name generally indicates what are its non-optional arguments, but not what are its optional ones. This is why it is safer and generally cleaner to name optional arguments.[4](chap65.xhtml#fn-footnote_22_note)

### Many parameters with the same type

As we said, when parameters have different types, we are generally safe from placing an argument at an incorrect position. There is no such freedom when some parameters have the same type. 

``` kotlin
fun sendEmail(to: String, message: String) { /*...*/ }
```

With a function like this, it is good to clarify arguments using names:

``` kotlin
sendEmail(
   to = "contact@kt.academy",
   message = "Hello, ..."
)
```

### Parameters of function type

Finally, we should treat parameters with function types specially. There is one special position for such parameters in Kotlin: the last position. Sometimes a function name describes an argument of a function type. For instance, when we see `repeat`, we expect that a lambda after that is the block of code that should be repeated. When you see `thread`, it is intuitive that the block after that is the body of this new thread. Such names only describe the function used at the last position. 

``` kotlin
thread {
   // ...
}
```

All other arguments with function types should be named because it is easy to misinterpret them. For instance, take a look at this simple view DSL:

``` kotlin
val view = linearLayout {
   text("Click below")
   button({ /* 1 */ }, { /* 2 */ })
}
```

Which function is a part of this builder and which one is an on-click listener? We should clarify it by naming the listener and moving builder outside of arguments:

``` kotlin
val view = linearLayout {
   text("Click below")
   button(onClick = { /* 1 */ }) {
      /* 2 */
   }
}
```

Multiple optional arguments of a function type can be especially confusing:

``` kotlin
fun call(before: ()->Unit = {}, after: ()->Unit = {}){
   before()
   print("Middle")
   after()
}

call({ print("CALL") }) // CALLMiddle
call { print("CALL") }  // MiddleCALL
```

To prevent such situations, when there is no single argument of a function type with special meaning, name them all:

``` kotlin
call(before = { print("CALL") }) // CALLMiddle
call(after = { print("CALL") })  // MiddleCALL
```

This is especially true for reactive libraries. For instance, in RxJava when we subscribe to an `Observable`, we can set functions that should be called:

- on every received item
- in case of error,
- after the observable is finished. 

In Java I’ve often seen people setting them up using lambda expressions, and specifying in comments which method each lambda expression is:

``` kotlin
// Java
observable.getUsers()
       .subscribe((List<User> users) -> { // onNext
           // ...
       }, (Throwable throwable) -> { // onError
           // ...
       }, () -> { // onCompleted
           // ...
       });
```

In Kotlin we can make a step forward and use named arguments instead:

``` kotlin
observable.getUsers()
   .subscribeBy(
       onNext = { users: List<User> ->
           // ...
       },
       onError = { throwable: Throwable ->
           // ...
       },
       onCompleted = {
           // ...
       })
```

Notice that I changed function name from `subscribe` to `subscribeBy`. It is because `RxJava` is written in Java and **we cannot use named arguments when we call Java functions**. It is because Java does not preserve information about function names. To be able to use named arguments we often need to make our Kotlin wrappers for those functions (extension functions that are alternatives to those functions). 

### Summary

Named arguments are not only useful when we need to skip some default values. They are important information for developers reading our code, and they can improve the safety of our code. We should consider them especially when we have more parameters with the same type (or with functional types) and for optional arguments. When we have multiple parameters with functional types, they should almost always be named. An exception is the last function argument when it has a special meaning, like in DSL.