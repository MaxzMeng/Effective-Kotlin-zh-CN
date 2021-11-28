## Item 44: Avoid member extensions

When we define an extension function to some class, it is not added to this class as a member. An extension function is just a different kind of function that we call on the first argument that is there, called a receiver. Under the hood, extension functions are compiled to normal functions, and the receiver is placed as the first parameter. For instance, the following function:

``` kotlin
fun String.isPhoneNumber(): Boolean =
   length == 7 && all { it.isDigit() }
```

Under the hood is compiled to a function similar to this one:

``` kotlin
fun isPhoneNumber(`$this`: String): Boolean =
   `$this`.length == 7 && `$this`.all { it.isDigit() }
```

One of the consequences of how they are implemented is that we can have member extensions or even define extensions in interfaces:

``` kotlin
interface PhoneBook {
   fun String.isPhoneNumber(): Boolean
}

class Fizz: PhoneBook {
   override fun String.isPhoneNumber(): Boolean =
        length == 7 && all { it.isDigit() }
}
```

Even though it is possible, there are good reasons to avoid defining member extensions (except for DSLs). **Especially, do not define extension as members just to restrict visibility**.

``` kotlin
// Bad practice, do not do this
class PhoneBookIncorrect {
   // ...  

   fun String.isPhoneNumber() =
     length == 7 && all { it.isDigit() }
}
```

One big reason is that it does not really restrict visibility. It only makes it more complicated to use the extension function since the user would need to provide both the extension and dispatch receivers:

``` kotlin
PhoneBookIncorrect().apply { "1234567890".test() }
```

**You should restrict the extension visibility using a visibility modifier and not by making it a member.**

``` kotlin
// This is how we limit extension functions visibility
class PhoneBookCorrect {
   // ...  
}

private fun String.isPhoneNumber() = 
     length == 7 && all { it.isDigit() }
```

There are a few good reasons why we prefer to avoid member extensions:

- Reference is not supported:

``` kotlin
val ref = String::isPhoneNumber
val str = "1234567890"
val boundedRef = str::isPhoneNumber

val refX = PhoneBookIncorrect::isPhoneNumber // ERROR
val book = PhoneBookIncorrect()
val boundedRefX = book::isPhoneNumber // ERROR
```

- Implicit access to both receivers might be confusing: 

``` kotlin
class A {
   val a = 10
}
class B {
   val a = 20
   val b = 30
  
   fun A.test() = a + b // Is it 40 or 50?
}
```

- When we expect an extension to modify or reference a receiver, it is not clear if we modify the extension or dispatch receiver (the class in which the extension is defined):

``` kotlin
class A {
   //...
}
class B {
   //...
  
   fun A.update() = ... // Does it update A or B?
}
```

- For less experienced developers it might be counterintuitive or scary to see member extensions. 

To summarize, if there is a good reason to use a member extension, it is fine. Just be aware of the downsides and generally try to avoid it. To restrict visibility, use visibility modifiers. Just placing an extension in a class does not limit its use from outside.