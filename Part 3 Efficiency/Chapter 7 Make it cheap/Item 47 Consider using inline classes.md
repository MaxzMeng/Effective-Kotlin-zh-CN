## Item 47: Consider using inline classes

Not only functions can be inlined, but also objects holding a single value can be replaced with this value. Such possibility was introduced as experimental in Kotlin 1.3, and to make it possible, we need to place the `inline` modifier before a class with a single primary constructor property:

``` kotlin
inline class Name(private val value: String) {
   // ...
}
```

Such a class will be replaced with the value it holds whenever possible:

``` kotlin
// Code
val name: Name = Name("Marcin")

// During compilation replaced with code similar to:
val name: String = "Marcin"
```

Methods from such a class will be evaluated as static methods:

``` kotlin
inline class Name(private val value: String) {
   // ...

   fun greet() {
       print("Hello, I am $value")
   }
}

// Code
val name: Name = Name("Marcin")
name.greet()

// During compilation replaced with code similar to:
val name: String = "Marcin"
Name.`greet-impl`(name)
```

We can use inline classes to make a wrapper around some type (like `String` in the above example) with no performance overhead (*Item 45: Avoid unnecessary object creation*). Two especially popular uses of inline classes are:

- To indicate a unit of measure
- To use types to protect user from misuse

Let’s discuss them separately.

### Indicate unit of measure

Imagine that you need to use a method to set up timer:

``` kotlin
interface Timer {
   fun callAfter(time: Int, callback: ()->Unit)
}
```

What is this `time`? Might be time in milliseconds, seconds, minutes… it is not clear at this point and it is easy to make a mistake. A serious mistake. One famous example of such a mistake is Mars Climate Orbiter that crashed into Mars atmosphere. The reason behind that was that the software used to control it was developed by an external company and it produced outputs in different units of measure than the ones expected by NASA. It produced results in pound-force seconds (lbf·s), while NASA expected newton-seconds (N·s). The total cost of the mission was 327.6 million USD and it was a complete failure. As you can see, confusion of measurement units can be really expensive. 

One common way for developers to suggest a unit of measure is by including it in the parameter name:

``` kotlin
interface Timer {
   fun callAfter(timeMillis: Int, callback: ()->Unit)
}
```

It is better but still leaves some space for mistakes. The property name is often not visible when a function is used. Another problem is that indicating the type this way is harder when the type is returned. In the example below, time is returned from `decideAboutTime` and its unit of measure is not indicated at all. It might return time in minutes and time setting would not work correctly then. 

``` kotlin
interface User {
   fun decideAboutTime(): Int
   fun wakeUp()
}

interface Timer {
   fun callAfter(timeMillis: Int, callback: ()->Unit)
}

fun setUpUserWakeUpUser(user: User, timer: Timer) {
   val time: Int = user.decideAboutTime()
   timer.callAfter(time) {
       user.wakeUp()
   }
}
```

We might introduce the unit of measure of the returned value in the function name, for instance by naming it `decideAboutTimeMillis`, but such a solution is rather rare as it makes a function longer every time we use it, and it states this low-level information even when we don’t need to know about it. 

A better way to solve this problem is to introduce stricter types that will protect us from misusing more generic types, and to make them efficient we can use inline classes:

``` kotlin
inline class Minutes(val minutes: Int) {
   fun toMillis(): Millis = Millis(minutes * 60 * 1000)
   // ...
}

inline class Millis(val milliseconds: Int) {
   // ...
}

interface User {
   fun decideAboutTime(): Minutes
   fun wakeUp()
}

interface Timer {
   fun callAfter(timeMillis: Millis, callback: ()->Unit)
}

fun setUpUserWakeUpUser(user: User, timer: Timer) {
   val time: Minutes = user.decideAboutTime()
   timer.callAfter(time) { // ERROR: Type mismatch
       user.wakeUp()
   }
}
```

This would force us to use the correct type:

``` kotlin
fun setUpUserWakeUpUser(user: User, timer: Timer) {
   val time = user.decideAboutTime()
   timer.callAfter(time.toMillis()) {
       user.wakeUp()
   }
}
```

It is especially useful for metric units, as in frontend, we often use a variety of units like pixels, millimeters, dp, etc. To support object creation, we can define DSL-like extension properties (and you can make them inline functions):

``` kotlin
inline val Int.min 
    get() = Minutes(this)

inline val Int.ms 
    get() = Millis(this)

val timeMin: Minutes = 10.min
```

### Protect us from type misuse

In SQL databases, we often identify elements by their IDs, which are all just numbers. Therefore, let’s say that you have a student grade in a system. It will probably need to reference the id of a student, teacher, school etc.:

``` kotlin
@Entity(tableName = "grades")
class Grades(
   @ColumnInfo(name = "studentId") 
   val studentId: Int,
   @ColumnInfo(name = "teacherId") 
   val teacherId: Int,
   @ColumnInfo(name = "schoolId") 
   val schoolId: Int,
   // ...
)
```

The problem is that it is really easy to later misuse all those ids, and the typing system does not protect us because they are all of type `Int`. The solution is to wrap all those integers into separate inline classes:

``` kotlin
inline class StudentId(val studentId: Int)
inline class TeacherId(val teacherId: Int)
inline class SchoolId(val studentId: Int)

@Entity(tableName = "grades")
class Grades(
   @ColumnInfo(name = "studentId") 
   val studentId: StudentId,
   @ColumnInfo(name = "teacherId") 
   val teacherId: TeacherId,
   @ColumnInfo(name = "schoolId") 
   val schoolId: SchoolId,
   // ...
)
```

Now those id uses will be safe, and at the same time, the database will be generated correctly because during compilation, all those types will be replaced with `Int` anyway. This way, inline classes allow us to introduce types where they were not allowed before, and thanks to that, we have safer code with no performance overhead. 

### Inline classes and interfaces

Inline classes just like other classes can implement interfaces. Those interfaces could let us properly pass time in any unit of measure we want. 

``` kotlin
interface TimeUnit {
   val millis: Long
}

inline class Minutes(val minutes: Long): TimeUnit {
   override val millis: Long get() = minutes * 60 * 1000
   // ...
}

inline class Millis(val milliseconds: Long): TimeUnit {
   override val millis: Long get() = milliseconds
}

fun setUpTimer(time: TimeUnit) {
   val millis = time.millis
   //...
}

setUpTimer(Minutes(123))
setUpTimer(Millis(456789))
```

The catch is that when an object is used through an interface, it cannot be inlined. Therefore in the above example there is no advantage to using inline classes, since wrapped objects need to be created to let us present a type through this interface. **When we present inline classes through an interface, such classes are not inlined.**

### Typealias

Kotlin typealias lets us create another name for a type:

``` kotlin
typealias NewName = Int
val n: NewName = 10
```

Naming types is a useful capability used especially when we deal with long and repeatable types. For instance, it is popular practice to name repeatable function types:

``` kotlin
typealias ClickListener = 
   (view: View, event: Event) -> Unit

class View {
   fun addClickListener(listener: ClickListener) {}
   fun removeClickListener(listener: ClickListener) {}
   //...
}
```

What needs to be understood though is that **typealiases do not protect us in any way from type misuse**. They are just adding a new name for a type. If we would name `Int` as both `Millis` and `Seconds`, we would make an illusion that the typing system protects us while it does not:

``` kotlin
typealias Seconds = Int
typealias Millis = Int

fun getTime(): Millis = 10
fun setUpTimer(time: Seconds) {}

fun main() {
   val seconds: Seconds = 10
   val millis: Millis = seconds // No compiler error
  
   setUpTimer(getTime())
}
```

In the above example it would be easier to find what is wrong without type aliases. This is why they should not be used this way. To indicate a unit of measure, use a parameter name or classes. A name is cheaper, but classes give better safety. When we use inline classes, we take the best from both options - it is both cheap and safe. 

### Summary

Inline classes let us wrap a type without performance overhead. Thanks to that, we improve safety by making our typing system protect us from value misuse. If you use a type with unclear meaning, especially a type that might have different units of measure, consider wrapping it with inline classes.