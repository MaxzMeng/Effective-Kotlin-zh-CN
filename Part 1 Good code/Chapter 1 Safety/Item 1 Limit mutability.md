In Kotlin, we design programs in modules and each of them is composed of different kinds of elements such as classes, objects, functions, type aliases and top-level properties. Some of those elements can hold state, for instance by having read-write property var or by composing a mutable object:

```kotlin
var a = 10
val list: MutableList<Int> = mutableListOf()
```

When an element holds state, the way it behaves depends not only on how you use it, but also on its history. A typical example of a class with a state is a bank account that has some money balance:

```kotlin
 class BankAccount {
    var balance = 0.0
       private set

    fun deposit(depositAmount: Double) {
        balance += depositAmount
    }

    @Throws(InsufficientFunds::class)
    fun withdraw(withdrawAmount: Double) {
        if (balance < withdrawAmount) {
            throw InsufficientFunds()
        }
        balance -= withdrawAmount
    }
 }

 class InsufficientFunds : Exception()

 val account = BankAccount()
 println(account.balance) // 0.0
 account.deposit(100.0)
 println(account.balance) // 100.0
 account.withdraw(50.0)
 println(account.balance) // 50.0
```

Here BankAccount has a state that represents how much money is present on that account. Holding state is a double-edged sword. On one hand it is very useful because it makes it possible to represent elements changing over time, but on the other hand state management is hard, because:

1. It is harder to understand and debug a program with many mutating points. The relationship between these mutations needs to be understood, and it is harder to track how they changed when there are more of them. A class with many mutating points that depend on each other is often really hard to understand and to modify. It is especially problematic in case of unexpected situations or errors.

2. Mutability makes it harder to reason about the code. State of immutable element is clear. Mutable state is much harder to comprehend. It is harder to reason what is its value as it might change at any point and just because we checked in some moment it doesn’t mean it is still the same.

3. It requires proper synchronization in multithreaded programs.Every mutation is a potential conflict.

4. Mutable elements are harder to test. We need to test every possible state, and the more mutability, the more states there are to test. What is more, the number of states we need to test generally grows exponentially with the number of mutation points in the same object or file, as we need to test all different combinations of possible states.
5. When state mutates, often some other classes need to be notified about this change. For instance, when we add a mutable element to a sorted list, once the element is changed, we need to sort this list again.

Problems with state consistency and the growing complexity of the project with more mutation points are familiar for developers working in bigger teams. Let’s see an example of how hard it is to manage shared state. Take a look at the below snippet⁶. It shows multiple threads trying to modify the same property, however because of conflicts some of those operations will be lost.

```kotlin
 var num = 0
 for (i in 1..1000) {
    thread {
        Thread.sleep(10)
        num += 1
    }
 }
 Thread.sleep(5000)
 print(num) // Very unlikely to be 1000
 // Every time a different number
```

When we use Kotlin coroutines, there are less conflicts because less threads are involved, but they still occur:

```kotlin
suspend fun main() {
    var num = 0
    coroutineScope {
        for (i in 1..1000) {
            launch {
                delay(10)
                num += 1
            }
        }
    }
    print(num) // Every time a different number
}
```

In real-life projects, we generally cannot just lose some operations, and so we need to implement proper synchronization like the one presented below. Although implementing proper synchronization is hard, and the more mutation points we have, the harder it is. Limiting mutability does help.

```kotlin
 val lock = Any()
 var num = 0
 for (i in 1..1000) {
    thread {
        Thread.sleep(10)
        synchronized(lock) {
            num += 1
        }
    }
 }
 Thread.sleep(1000)
 print(num) // 1000
```

The drawbacks of mutability are so numerous that there are languages that do not allow state mutation at all. These are purely functional languages. One well-known example is Haskell. Such languages are rarely used for mainstream development though, since it’s very hard to do programming with so limited mutability. Mutating state is a very useful way to represent the state of real-world systems. I recommend using mutability, but do it sparingly and wisely decide where our mutating points should be. The good news is that Kotlin supports limiting mutability well.

#### Limiting mutability in Kotlin

Kotlin is designed to support limiting mutability. It is easy to make immutable objects or to keep properties immutable. It is a result of many features and characteristics of this language, but the most important ones are:

- Read-only properties val

- Separation between mutable and read-only collections

- copy in data classes

Let’s discuss them one by one.

#### Read-only properties val

In Kotlin we can make each property read-only val (like value) or read-write var (like variable). Read-only properties val do not allow setting:

```kotlin
val a = 10
a = 20 // ERROR
```

Notice though that read-only properties are not necessarily immutable nor final. A read-only property can hold a mutable object:

```kotlin
val list = mutableListOf(1,2,3) 
list.add(4)

print(list) // [1, 2, 3, 4]
```

A read-only property can also be defined using a custom getter that might depend on another property:

```kotlin
var name: String = "Marcin"
var surname: String = "Moskała"
val fullName 
    get() = "$name $surname"

fun main() {
   println(fullName) // Marcin Moskała
   name = "Maja"
   println(fullName) // Maja Moskała
}
```

Notice that it is possible because when we define a custom getter, it will be called every time we ask for the value.

```kotlin
 fun calculate(): Int {
    print("Calculating... ")
    return 42
 }
 
 val fizz = calculate() // Calculating...
 val buzz
    get() = calculate()
 
 fun main() {
    print(fizz) // 42
    print(fizz) // 42
    print(buzz) // Calculating... 42
    print(buzz) // Calculating... 42
 }
```

This trait, that properties in Kotlin are encapsulated by default and they can have custom accessors (getters and setters) is very important in Kotlin because it gives us flexibility when we change or define our API. It will be described in detail in Item 16: Properties should represent state, not behavior. The core idea though is that val do not offer mutation points because it is only a getter under the hood when var is both getter and setter. That’s why we can override val with var:

```kotlin
interface Element {
   val active: Boolean
}

class ActualElement: Element {
   override var active: Boolean = false
}
```

Values of read-only properties val can change, but such properties do not offer a mutation point which is the main source of problems when we need to synchronize or reason about a program. This is why we generally prefer val over var.

Although remember that val doesn’t mean immutable. It can be defined by getter or delegate. This fact gives us more freedom to change. Though when we don’t need that, final properties should be preferred. It is easier to reason about them as they have the state stated next to their definition. They are also better supported in Kotlin. For instance, they can be smart-casted:

```kotlin
val name: String? = "Márton"
val surname: String = "Braun"

val fullName: String?
   get() = name?.let { "$it $surname" }

val fullName2: String? = name?.let { "$it $surname" }

fun main() {
   if (fullName != null) {
       println(fullName.length) // ERROR
   }

   if (fullName2 != null) {
       println(fullName2.length) // Márton Braun
   }
}
```

Smart cast is impossible for fullName because it is defined using getter, so it might give a different value during check and different later during use (for instance, if some other thread would set name). Non-local properties can be smart-casted only when they are final and do not have custom getter.

#### Separation between mutable and read-only collections

Similarly, as Kotlin separates read-write and read-only properties, Kotlin separates read-write and read-only collections. This is achieved thanks to the way the hierarchy of collections was designed. Take a look at the diagram presenting collections hierarchy in Kotlin. On the left side, you can see the Iterable, Collection, Set, and List interfaces that are read-only. This means that they do not have any methods that would allow modification. On the right side, you can see the MutableIterable, MutableCollection, MutableSet, and MutableList interfaces that represent mutable collections. Notice that each mutable interface extends the corresponding read-only interface, and adds methods that allow mutation. This is similar to how properties work. A read-only property means just a getter, while a read-write property means both a getter and a setter.

![The hierarchy of collection interfaces in Kotlin and actual objects that can be used on Kotlin/JVM. On the left side interfaces are read-only. On the right side collections and interfaces are mutable.](../../assets/chapter1/chapter1-1.png)

Read-only collections are not necessarily immutable. Very often they are mutable, but they cannot be mutated because they are hidden behind read-only interfaces. For instance, the Iterable<T>.map and Iterable<T>.filter functions return ArrayList, which is a mutable list, as a List, which is a read-only interface. In the below snippet you can see a simplified implementation of Iterable<T>.map from stdlib.

```kotlin
inline fun <T, R> Iterable<T>.map(
   transformation: (T) -> R
): List<R> {
   val list = ArrayList<R>()
   for (elem in this) {
       list.add(transformation(elem))
   }
   return list
}
```

The design choice to make these collection interfaces read-only instead of truly immutable is very important. It gives us much more freedom. Under the hood, any actual collection can be returned as long as it satisfies the interface. Therefore, we can use platform-specific collections.

The safety of this approach is close to the one achieved from having immutable collections. The only risk is when a developer tries to “hack the system” and performs down-casting. This is something that should never be allowed in Kotlin projects. We should be able to trust that when we return a list as read-only, it will be used only to read it. This is part of the contract. More about it on Part 2 of this book.

Down-casting collections is not only breaking their contract and depending on implementation instead of abstraction as we should, but it is also insecure and can lead to surprising consequences. Take a look at this code:

```kotlin
val list = listOf(1,2,3)

// DON’T DO THIS!
if (list is MutableList) {
   list.add(4)
}
```

The result of this operation is platform-specific. On the JVM listOf returns an instance of Arrays.ArrayList that implements Java List interface. This Java List interface has methods like add or set, and so it translates to the Kotlin MutableList interface. However, Arrays.ArrayList does not implement some of those operations. This is why the result of the above code is the following:

```kotlin
Exception in thread “main” java.lang.UnsupportedOperationException
at java.util.AbstractList.add(AbstractList.java:148)
at java.util.AbstractList.add(AbstractList.java:108)
```

Though there is no guarantee how this will behave in a year from now. Underlying collections might change. They might be replaced with truly immutable collections implemented in Kotlin and not implementing MutableList at all. Nothing is guaranteed. This is why down-casting read-only collections to mutable should never take place in Kotlin. If you need to change from read-only to mutable, you should use List.toMutableList function, which creates a copy that you can then modify:

```kotlin
val list = listOf(1, 2, 3)

val mutableList = list.toMutableList()
mutableList.add(4)
```

This way does not break any contract, and it is also safer for us as we can feel safe that when we expose something as List it won’t be modified from outside.

#### Copy in data classes

There are many reasons to prefer immutable objects - objects that do not change their internal state, like String or Int. In addition to the already named reasons why we generally prefer less mutability, immutable objects have their own advantages:

1. They are easier to reason about since their state stays the same once they are created.
2. Immutability makes it easier to parallelize the program as there are no conflicts among shared objects.
3. References to immutable objects can be cached as they are not going to change.
4. We do not need to make defensive copies on immutable objects. When we do copy immutable objects, we do not need to make it a deep copy.
5. Immutable objects are the perfect material to construct other objects. Both mutable and immutable. We can still decide where mutability takes place, and it is easier to operate on immutable objects.
6. We can add them to set or use them as keys in maps, in opposition to mutable objects that shouldn’t be used this way. This is because both those collections use hash table under the hood in Kotlin/JVM, and when we modify elements already classified to a hash table, its classification might not be correct anymore and we won’t be able to find it. This problem will be described in detail in Item 41: Respect the contract of hashCode. We have a similar issue when a collection is sorted.

```kotlin
val names: SortedSet<FullName> = TreeSet()
val person = FullName("AAA", "AAA")
names.add(person)
names.add(FullName("Jordan", "Hansen"))
names.add(FullName("David", "Blanc"))

print(s) // [AAA AAA, David Blanc, Jordan Hansen]
print(person in names) // true

person.name = "ZZZ"
print(names) // [ZZZ AAA, David Blanc, Jordan Hansen]
print(person in names) // false”
```

At the last check, collection returned false even though that person is in this set. It couldn’t be found because it is in an incorrect position.

As you can see, mutable objects are more dangerous and less predictable. On the other hand, the biggest problem of immutable objects is that data sometimes needs to change. The solution is that the immutable objects should have methods that produce an object after some change. For instance, Int is immutable, and it has many methods like plus or minus that do not modify it but instead return a new Int after this operation. Iterable is read-only, and collection processing functions like map or filter do not modify it, but instead return a new collection. The same can be applied to our immutable objects. For instance, let’s say that we have an immutable class User and we need to allow its surname to change. We can support it with a withSurname method that produces a copy with a particular property changed:

```kotlin
class User(
   val name: String,
   val surname: String
) {
   fun withSurname(surname: String) = User(name, surname)
}

var user = User("Maja", "Markiewicz")
user = user.withSurname("Moskała")
print(user) // User(name=Maja, surname=Moskała)
```

Writing such functions is possible, but also tedious if we need one for every property. Here comes the data modifier to the rescue. One of the methods it generates is copy. It creates a new instance where all primary constructor properties are the same as in the previous one by default. New values can be specified as well. copy together with other methods generated by data modifier are described in detail in Item 37: Use data modifier to represent a bundle of data. Here is a simple example showing how it works:

```kotlin
data class User(
   val name: String,
   val surname: String
)

var user = User("Maja", "Markiewicz")
user = user.copy(surname = "Moskała")
print(user) // User(name=Maja, surname=Moskała)
```

This is an elegant and universal solution that supports making data model classes immutable. Surely, this way is less efficient than just using a mutable object instead, but it has all described advantages of immutable objects and should be preferred by default.

#### Different kinds of mutation points

Let’s say that we need to represent a mutating list. There are two ways we can achieve that. Either by using a mutable collection or by using read-write property var:

```kotlin
val list1: MutableList<Int> = mutableListOf() 
var list2: List<Int> = listOf()
```

Both properties can be modified, but in different ways:

```kotlin
list1.add(1) 
list2 = list2 + 1
```

Both of these ways can be replaced with the plus-assign operator as well, but each of them is translated into a different behavior:

```kotlin
list1 += 1 // Translates to list1.plusAssign(1) 
list2 += 1 // Translates to list2 = list2.plus(1)
```

Both those ways are correct and they both have their pros and cons. They both have a single mutating point, but it is located in a different place. In the first one mutation takes place on the concrete list implementation. We might depend on the fact that it has proper synchronization in case of multithreading, but such an assumption is also dangerous since it is not really guaranteed. In the second one, we need to implement the synchronization ourselves, but the overall safety is better because the mutating point is only a single property. Though, in case of a lack of synchronization, remember that we can still lose some elements:

```kotlin
ar list = listOf<Int>()
for (i in 1..1000) {
   thread {
       list = list + i
   }
}
Thread.sleep(1000)
print(list.size) // Very unlikely to be 1000, 
// every time a different number, like for instance 911
```

Using a mutable property instead of a mutable list allows us to track how this property changes when we define a custom setter or using a delegate (which is using a custom setter). For instance, when we use an observable delegate, we can log every change of a list:

```kotlin
var names by Delegates.observable(listOf<String>()) { _, old, new ->
   println("Names changed from $old to $new")
}

names += "Fabio" 
// Names changed from [] to [Fabio]
names += "Bill"   
// Names changed from [Fabio] to [Fabio, Bill]”
```

To make this possible for a mutable collection, we would need a special observable implementation of the collection. For read-only collections on mutable properties, it is also easier to control how they change - there is only a setter instead of multiple methods mutating this object, and we can make it private:

```kotlin
var announcements = listOf<Announcement>()
   private set
```

In short, using mutable collections is a slightly faster option, but using a mutable property instead gives us more control over how the object is changing.

Notice that the worst solution is to have both a mutating property and a mutable collection:

```kotlin
// Don’t do that
var list3 = mutableListOf<Int>()
```

We would need to synchronize both ways it can mutate (by property change and by internal state change). Also, changing it using plus-assign is impossible because of ambiguity:

![](../../assets/chapter1/chapter1-2.png)

The general rule is that one should not create unnecessary ways to mutate state. Every way to mutate state is a cost. It needs to be understood and maintained. We prefer to limit mutability.

#### Do not leak mutation points

It is an especially dangerous situation when we expose a mutable object that makes up state. Take a look at this example:

```kotlin
data class User(val name: String)

class UserRepository {
   private val storedUsers: MutableMap<Int, String> = 
       mutableMapOf()

   fun loadAll(): MutableMap<Int, String> {
       return storedUsers
   }

   //...
}
```

One could use loadAll to modify UserRepository private state:

```kotlin
val userRepository = UserRepository()

val storedUsers = userRepository.loadAll()
storedUsers[4] = "Kirill"
//...

print(userRepository.loadAll()) // {4=Kirill}
```

It is especially dangerous when such modifications are accidental. There are two ways how we can deal with that. The first one is copying returned mutable objects. We call that defensive copying. This can be a useful technique when we deal with a standard objects and here copy generated by data modifier can be really helpful:

```kotlin
class UserHolder {
    private val user: MutableUser()

    fun get(): MutableUser {
        return user.copy()
    }

    //...
}
```

Though whenever possible we prefer limiting mutability, and for collections we can do that by upcasting those objects to their read-only supertype:

```kotlin
data class User(val name: String)

class UserRepository {
   private val storedUsers: MutableMap<Int, String> = 
       mutableMapOf()

   fun loadAll(): Map<Int, String> {
       return storedUsers
   }

   //...
}
```

#### Summary

In this chapter we’ve learned why it is important to limit mutability and to prefer immutable objects. We’ve seen that Kotlin gives us a lot of tools that support limiting mutability. We should use them to limit mutation points. Simple rules are:

- Prefer val over var.
- Prefer an immutable property over a mutable one.
- Prefer objects and classes that are immutable over mutable.
- If you need them to change, consider making them immutable data classes, and using copy.
-  When you hold state, prefer read-only over mutable collections.
- Design your mutation points wisely and do not produce unnecessary ones.
-  Do not expose mutable objects.

There are some exceptions to these rules. Sometimes we prefer a mutable object because they are more efficient. Such optimizations should be preferred only in the performance critical parts of our code (Part 3: Efficiency) and when we use them, we need to remember that mutability requires more attention when we prepare it for multithreading. The baseline is that we should limit mutability.