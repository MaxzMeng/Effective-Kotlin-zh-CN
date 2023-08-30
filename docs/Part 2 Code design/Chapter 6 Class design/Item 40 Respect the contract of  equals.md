## Item 40: Respect the contract of `equals`

In Kotlin, every object extends `Any`, which has a few methods with well-established contracts. These methods are:

- `equals`
- `hashCode`
- `toString`

Their contract is described in their comments and elaborated in the official documentation, and as I described in *Item 32: Respect abstraction contracts*, every subtype of a type with a set contract should respect this contract. These methods have an important position in Kotlin, as they have been defined since the beginning of Java, and therefore many objects and functions depend on their contract. Breaking their contract will often lead to some objects or functions not working properly. This is why in the current and next items we will talk about overriding these functions and about their contracts. Let’s start with `equals`. 

### Equality

In Kotlin, there are two types of equality:

- Structural equality - checked by the `equals` method or `==` operator (and its negated counterpart `!=`) which is based on the `equals` method. `a == b` translates to `a.equals(b)` when `a` is not nullable, or otherwise to `a?.equals(b) ?: (b === null)`.
- Referential equality - checked by the `===` operator (and its negated counterpart `!==`), returns `true` when both sides point to the same object. 

Since `equals` is implemented in `Any`, which is the superclass of every class, we can check the equality of any two objects. Although using operators to check equality is not allowed when objects are not of the same type:

``` kotlin
open class Animal
class Book
Animal() == Book()  // Error: Operator == cannot be 
// applied to Animal and Book
Animal() === Book() // Error: Operator === cannot be 
// applied to Animal and Book
```

Objects either need to have the same type or one needs to be a subtype of another:

``` kotlin
class Cat: Animal()
Animal() == Cat()  // OK, because Cat is a subclass of 
// Animal
Animal() === Cat() // OK, because Cat is a subclass of 
// Animal
```

It is because it does not make sense to check equality of two objects of a different type. It will get clear when we will explain the contract of equals. 

### Why do we need equals?

The default implementation of `equals` coming from `Any` checks if another object is exactly the same instance. Just like referential equality (`===`). It means that every object by default is unique:

``` kotlin
class Name(val name: String)
val name1 = Name("Marcin")
val name2 = Name("Marcin")
val name1Ref = name1

name1 == name1 // true
name1 == name2 // false
name1 == name1Ref // true

name1 === name1 // true
name1 === name2 // false
name1 === name1Ref // true
```

Such behavior is useful for many objects. It is perfect for active elements, like a database connection, a repository, or a thread. However, there are objects where we need to represent equality differently. A popular alternative is a data class equality that checks if all primary constructor properties are equal:

``` kotlin
data class FullName(val name: String, val surname: String)
val name1 = FullName("Marcin", "Moskała")
val name2 = FullName("Marcin", "Moskała")
val name3 = FullName("Maja", "Moskała")
  
name1 == name1 // true
name1 == name2 // true, because data are the same
name1 == name3 // false
  
name1 === name1 // true
name1 === name2 // false
name1 === name3 // false
```

Such behavior is perfect for classes that are represented by the data they hold, and so we often use the data modifier in data model classes or in other data holders. 

Notice that data class equality also helps when we need to compare some but not all properties. For instance when we want to skip cache or other redundant properties. Here is an example of an object representing date and time having properties `asStringCache` and `changed` that should not be compared by equality check: 

``` kotlin
class DateTime(
   /** The millis from 1970-01-01T00:00:00Z */
   private var millis: Long = 0L,
   private var timeZone: TimeZone? = null
) {
   private var asStringCache = ""
   private var changed = false

   override fun equals(other: Any?): Boolean =
       other is DateTime &&
               other.millis == millis &&
               other.timeZone == timeZone
  
   //...
}
```

The same can be achieved by using data modifier:

``` kotlin
data class DateTime(
   private var millis: Long = 0L,
   private var timeZone: TimeZone? = null
) {
   private var asStringCache = ""
   private var changed = false
  
   //...
}
```

Just notice that `copy` in such case will not copy those properties that are not declared in the primary constructor. Such behavior is correct only when those additional properties are truly redundant (the object will behave correctly when they will be lost).

Thanks to those two alternatives, default and data class equality, **we rarely need to implement equality ourselves in Kotlin**. Although there are cases where we need to implement equals ourselves. 

Another example is when just concrete property decides if two objects are equal. For instance, a `User` class might have an assumption that two users are equal when their `id` is equal. 

``` kotlin
class User(
   val id: Int,
   val name: String,
   val surname: String
) {
   override fun equals(other: Any?): Boolean =
       other is User && other.id == id

   override fun hashCode(): Int = id
}
```

As you can see, we implement `equals` ourselves when:

- We need its logic to differ from the default one
- We need to compare only a subset of properties
- We do not want our object to be a data class or properties we need to compare are not in the primary constructor

### The contract of equals

This is how `equals` is described in its comments (Kotlin 1.3.11, formatted):

Indicates whether some other object is “equal to” this one. Implementations must fulfill the following requirements:

- Reflexive: for any non-null value `x`, `x.equals(x)` should return `true`.
- Symmetric: for any non-null values `x` and `y`, `x.equals(y)` should return `true` if and only if `y.equals(x)` returns true.
- Transitive: for any non-null values `x`, `y`, and `z`, if `x.equals(y)` returns `true` and `y.equals(z)` returns `true`, then `x.equals(z)` should return `true`.
- Consistent: for any non-null values `x` and `y`, multiple invocations of `x.equals(y)` consistently return `true` or consistently return `false`, provided no information used in `equals` comparisons on the objects is modified.
- Never equal to null: for any non-null value `x`, `x.equals(null)` should return `false`.

Additionally, we expect `equals`, `toString` and `hashCode` to be fast. It is not a part of the official contract, but it would be highly unexpected to wait a few seconds to check if two elements are equal.

All those requirements are important. They are assumed from the beginning, also in Java, and so now many objects depend on those assumptions. Don’t worry if they sound confusing right now, we’ll describe them in detail.

- Object equality should be **reflexive**, meaning that `x.equals(x)` returns `true`. Sounds obvious, but this can be violated. For instance, someone might want to make a `Time` object that represents the current time, and compares milliseconds:

``` kotlin
// DO NOT DO THIS!
class Time(
   val millisArg: Long = -1,
   val isNow: Boolean = false
) {
   val millis: Long get() =
       if (isNow) System.currentTimeMillis() 
       else millisArg

   override fun equals(other: Any?): Boolean =
       other is Time && millis == other.millis
}

val now = Time(isNow = true)
now == now // Sometimes true, sometimes false   
List(100000) { now }.all { it == now }
// Most likely false
```

Notice that here the result is inconsistent, so it also violates the last principle. 

When an object is not equal to itself, it might not be found in most collections even if it is there when we check using the `contains` method. It will not work correctly in most unit tests assertions either. 

``` kotlin
val now1 = Time(isNow = true)
val now2 = Time(isNow = true)
assertEquals(now1, now2) 
// Sometimes passes, sometimes not
```

**When the result is not constant, we cannot trust it.** We can never be sure if the result is correct or is it just a result of inconsistency. 

How should we improve it? A simple solution is checking separately if the object represents the current time and if not, then whether it has the same timestamp. Though it is a typical example of tagged class, and as described in *Item 39: Prefer class hierarchies to tagged classes*, it would be even better to use class hierarchy instead:

``` kotlin
sealed class Time
data class TimePoint(val millis: Long): Time()
object Now: Time()
```

- Object equality should be **symmetric**, meaning that the result of `x == y` and `y == x` should always be the same. It can be easily violated when in our equality we accept objects of a different type. For instance, let’s say that we implemented a class to represent complex numbers and made its equality accept `Double`:

``` kotlin
class Complex(
   val real: Double,
   val imaginary: Double
) {
   // DO NOT DO THIS, violates symmetry
   override fun equals(other: Any?): Boolean {
       if (other is Double) { 
          return imaginary == 0.0 && real == other
       }
       return other is Complex &&
               real == other.real &&
               imaginary == other.imaginary
   }
}
```

The problem is that `Double` does not accept equality with `Complex`. Therefore the result depends on the order of the elements:

``` kotlin
Complex(1.0, 0.0).equals(1.0) // true
1.0.equals(Complex(1.0, 0.0)) // false
```

Lack of symmetry means, for instance, unexpected results on collections `contains` or on unit tests assertions.

``` kotlin
val list = listOf<Any>(Complex(1.0, 0.0))
list.contains(1.0) // Currently on the JVM this is false,
// but it depends on the collection’s implementation 
// and should not be trusted to stay the same
```

**When equality is not symmetric and it is used by another object, we cannot trust the result because it depends on whether this object compares `x` to `y` or `y` to `x`.** This fact is not documented and it is not a part of the contract as object creators assume that both should work the same (they assume symmetry). It can also change at any moment - creators during some refactorization might change the order. If your object is not symmetric, it might lead to unexpected and really hard to debug errors in your implementation. This is why when we implement `equals` we should always consider equality. 

The general solution is that we should not accept equality between different classes. I’ve never seen a case where it would be reasonable. Notice that in Kotlin similar classes are not equal to each other. 1 is not equal to 1.0, and 1.0 is not equal to 1.0F. Those are different types and they are not even comparable. Also, in Kotlin we cannot use the `==` operator between two different types that do not have a common superclass other than `Any`:

``` kotlin
Complex(1.0, 0.0) == 1.0 // ERROR
```

- Object equality should be **transitive**, meaning that for any non-null reference values `x`, `y`, and `z`, if `x.equals(y)` returns `true` and `y.equals(z)` returns `true`, then `x.equals(z)` should return `true`. The biggest problem with transitivity is when we implement different kinds of equality that check a different subtype of properties. For instance, let’s say that we have `Date` and `DateTime` defined this way:

``` kotlin
open class Date(
   val year: Int,
   val month: Int,
   val day: Int
) {
   // DO NOT DO THIS, symmetric but not transitive
   override fun equals(o: Any?): Boolean = when (o) {
       is DateTime -> this == o.date
       is Date -> o.day == day && o.month == month && 
o.year == year
       else -> false
   }

   // ...
}

class DateTime(
   val date: Date,
   val hour: Int,
   val minute: Int,
   val second: Int
): Date(date.year, date.month, date.day) {
   // DO NOT DO THIS, symmetric but not transitive
   override fun equals(o: Any?): Boolean = when (o) {
       is DateTime -> o.date == date && o.hour == hour && 
o.minute == minute && o.second == second
       is Date -> date == o
       else -> false
   }

   // ...
}
```

The problem with the above implementation is that when we compare two `DateTime`, we check more properties than when we compare `DateTime` and `Date`. Therefore two `DateTime` with the same day but a different time will not be equal to each other, but they’ll both be equal to the same `Date`. As a result, their relation is not transitive:

``` kotlin
val o1 = DateTime(Date(1992, 10, 20), 12, 30, 0)
val o2 = Date(1992, 10, 20)
val o3 = DateTime(Date(1992, 10, 20), 14, 45, 30)

o1 == o2 // true
o2 == o3 // true
o1 == o3 // false <- So equality is not transitive
```

Notice that here the restriction to compare only objects of the same type didn’t help because we’ve used inheritance. Such inheritance violates the *Liskov substitution principle* and should not be used. In this case, use composition instead of inheritance (*Item 36: Prefer composition over inheritance*). When you do, do not compare two objects of different types. These classes are perfect examples of objects holding data and representing them this way is a good choice:

``` kotlin
data class Date(
   val year: Int,
   val month: Int,
   val day: Int
)

data class DateTime(
   val date: Date,
   val hour: Int,
   val minute: Int,
   val second: Int
)

val o1 = DateTime(Date(1992, 10, 20), 12, 30, 0)
val o2 = Date(1992, 10, 20)
val o3 = DateTime(Date(1992, 10, 20), 14, 45, 30)

o1.equals(o2) // false
o2.equals(o3) // false
o1 == o3 // false

o1.date.equals(o2) // true
o2.equals(o3.date) // true
o1.date == o3.date // true
```

- Equality should be **consistent**, meaning that the method invoked on two objects should always return the same result unless one of those objects was modified. For immutable objects, the result should be always the same. In other words, we expect `equals` to be a pure function (do not modify the state of an object) for which result always depends only on input and state of its receiver. We’ve seen the `Time` class that violated this principle. This rule was also famously violated in `java.net.URL.equals()`.
- Never equal to null: for any non-null value `x`, `x.equals(null)` must return `false`. It is important because `null` should be unique and no object should be equal to it. 

### Problem with equals in URL

One example of a really poorly designed `equals` is the one from `java.net.URL`. Equality of two `java.net.URL` objects depends on a network operation as two hosts are considered equivalent if both hostnames can be resolved into the same IP addresses. Take a look at the following example:

``` kotlin
import java.net.URL

fun main() {
   val enWiki = URL("https://en.wikipedia.org/")
   val wiki = URL("https://wikipedia.org/")
   println(enWiki == wiki)
}
```

The result is not consistent. In normal conditions, it should print `true` because those two addresses are considered equal according to `equals` implementation, although if you have your internet turned off, it will print `false`. You can check it yourself. This is a big mistake! Equality should not be network dependent. 

Here are the most important problems with this solution:

- **This behavior is inconsistent.** For instance, two URLs could be equal when a network is available and unequal when it is not. Also, the network may change. The IP address for a given hostname varies over time and by the network. Two URLs could be equal on some networks and unequal on others.
- **The network may be slow and** **we expect** `equals` **and** `hashCode` **to be fast.** A typical problem is when we check if a URL is present in a list. Such an operation would require a network call for each element on the list. Also on some platforms, like Android, network operations are prohibited on the main thread. As a result, even adding to a set of URLs needs to be started on a separate thread. 
- **The defined behavior is known to be inconsistent with virtual hosting in HTTP.** Equal IP addresses do not imply equal content. Virtual hosting permits unrelated sites to share an IP address. This method could report two otherwise unrelated URLs to be equal because they’re hosted on the same server.

In Android, this problem was fixed in Android 4.0 (Ice Cream Sandwich). Since that release, URLs are only equal if their hostnames are equal. When we use Kotlin/JVM on other platforms, it is recommended to use `java.net.URI` instead of `java.net.URL`.

### Implementing equals

I recommend against implementing equals yourself unless you have a good reason. Instead, use default or data class equality. If you do need custom equality, always consider if your implementation is reflexive, symmetric, transitive, and consistent. Make such class final, or beware that subclasses should not change how equality behaves. It is hard to make custom equality and support inheritance at the same time. Some even say it is impossible[3](chap65.xhtml#fn-equals_and_contract). Data classes are final.