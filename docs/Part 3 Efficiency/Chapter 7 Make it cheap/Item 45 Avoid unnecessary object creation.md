## Item 45: Avoid unnecessary object creation

Object creation can sometimes be expensive and always costs something. This is why avoiding unnecessary object creation can be an important optimization. It can be done on many levels. For instance, in JVM it is guaranteed that a string object will be reused by any other code running in the same virtual machine that happens to contain the same string literal[1](chap65.xhtml#fn-java_spec):

``` kotlin
val str1 = "Lorem ipsum dolor sit amet"
val str2 = "Lorem ipsum dolor sit amet"
print(str1 == str2) // true
print(str1 === str2) // true
```

Boxed primitives (Integer, Long) are also reused in JVM when they are small (by default Integer Cache holds numbers in a range from -128 to 127).

``` kotlin
val i1: Int? = 1
val i2: Int? = 1
print(i1 == i2) // true
print(i1 === i2) // true, because i2 was taken from cache 
```

Reference equality shows that this is the same object. If we use number that is either smaller than -128 or bigger than 127 though, different objects will be produced:

``` kotlin
val j1: Int? = 1234
val j2: Int? = 1234
print(j1 == j2) // true
print(j1 === j2) // false
```

Notice that a nullable type is used to force `Integer` instead of `int` under the hood. When we use `Int`, it is generally compiled to the primitive `int`, but when we make it nullable or when we use it as a type argument, `Integer` is used instead. It is because primitive cannot be `null` and cannot be used as a type argument. Knowing that such mechanisms were introduced in the language, you might wonder how significant they are. Is object creation expensive?

### Is object creation expensive?

Wrapping something into an object has 3 parts of cost:

- **Objects take additional space.** In a modern 64-bit JDK, an object has a 12-byte header, padded to a multiple of 8 bytes, so the minimum object size is 16 bytes. For 32-bit JVMs, the overhead is 8 bytes. Additionally, object references take space as well. Typically, references are 4 bytes on 32bit platforms or on 64bit platforms up to -Xmx32G, and 8 bytes above 32Gb (-Xmx32G). Those are relatively small numbers, but they can add up to a significant cost. When we think about such small elements like integers, they make a difference. `Int` as a primitive fits in 4 bytes, when as a wrapped type on 64-bit JDK we mainly use today, it requires 16 bytes (it fits in the 4 bytes after the header) + its reference requires 4 or 8 bytes. In the end, it takes 5 or 6 times more space[2](chap65.xhtml#fn-measure).
- **Access requires an additional function call when elements are encapsulated.** That is again a small cost as function use is really fast, but it can add-up when we need to operate on a huge pool of objects. Do not limit encapsulation, avoid creating unnecessary objects especially in performance critical parts of your code. 
- **Objects need to be created.** An object needs to be created, allocated in the memory, a reference needs to be created, etc. It is also a really small cost, but it can add up. 

``` kotlin
class A
private val a = A()

// Benchmark result: 2.698 ns/op
fun accessA(blackhole: Blackhole) {
   blackhole.consume(a)
}

// Benchmark result: 3.814 ns/op
fun createA(blackhole: Blackhole) {
   blackhole.consume(A())
}

// Benchmark result: 3828.540 ns/op
fun createListAccessA(blackhole: Blackhole) {
   blackhole.consume(List(1000) { a })
}

// Benchmark result: 5322.857 ns/op
fun createListCreateA(blackhole: Blackhole) {
   blackhole.consume(List(1000) { A() })
}
```

By eliminating objects, we can avoid all three costs. By reusing objects, we can eliminate the first and the third one. Knowing that, you might start thinking about limiting the number of unnecessary objects in your code. Let’s see some ways we can do that.

### Object declaration

A very simple way to reuse an object instead of creating it every time is using object declaration (singleton). To see an example, let’s imagine that you need to implement a linked list. The linked list can be either empty, or it can be a node containing an element and pointing to the rest. This is how it can be implemented:

``` kotlin
sealed class LinkedList<T>

class Node<T>(
    val head: T, 
    val tail: LinkedList<T>
): LinkedList<T>()

class Empty<T>: LinkedList<T>()

// Usage
val list: LinkedList<Int> = 
    Node(1, Node(2, Node(3, Empty())))
val list2: LinkedList<String> = 
    Node("A", Node("B", Empty()))
```

One problem with this implementation is that we need to create an instance of `Empty` every time we create a list. Instead, we should just have one instance and use it universally. The only problem is the generic type. What generic type should we use? We want this empty list to be subtype of all lists. We cannot use all types, but we also don’t need to. A solution is that we can make it a list of `Nothing`. `Nothing` is a subtype of every type, and so `LinkedList<Nothing>` will be a subtype of every `LinkedList` once this list is covariant (`out` modifier). Making type arguments covariant truly makes sense here since the list is immutable and this type is used only in out positions (Item 24: Consider variance for generic types). So this is the improved code:

``` kotlin
sealed class LinkedList<out T>

class Node<out T>(
       val head: T,
       val tail: LinkedList<T>
) : LinkedList<T>()

object Empty : LinkedList<Nothing>()

// Usage
val list: LinkedList<Int> = 
    Node(1, Node(2, Node(3, Empty)))

val list2: LinkedList<String> = 
    Node("A", Node("B", Empty))
```

This is a useful trick that is often used, especially when we define immutable sealed classes. Immutable, because using it for mutable objects can lead to subtle and hard to detect bugs connected to shared state management. The general rule is that mutable objects should not be cached (*Item 1: Limit mutability*). Apart from object declaration, there are more ways to reuse objects. Another one is a factory function with a cache.

### Factory function with a cache

Every time we use a constructor, we have a new object. Though it is not necessarily true when you use a factory method. Factory functions can have cache. The simplest case is when a factory function always returns the same object. This is, for instance, how `emptyList` from stdlib is implemented:

``` kotlin
fun <T> emptyList(): List<T> = EmptyList
```

Sometimes we have a set of objects, and we return one of them. For instance, when we use the default dispatcher in the Kotlin coroutines library `Dispatchers.Default`, it has a pool of threads, and whenever we start anything using it, it will start it in one that is not in use. Similarly, we might have a pool of connections with a database. Having a pool of objects is a good solution when object creation is heavy, and we might need to use a few mutable objects at the same time. 

Caching can also be done for parameterized factory methods. In such a case, we might keep our objects in a map:

``` kotlin
private val connections: MutableMap<String, Connection> = 
    mutableMapOf<String, Connection>()

fun getConnection(host: String) =
    connections.getOrPut(host) { createConnection(host) }
```

Caching can be used for all pure functions. In such a case, we call it memoization. Here is, for instance, a function that calculates the Fibonacci number at a position based on the definition:

``` kotlin
private val FIB_CACHE: MutableMap<Int, BigInteger> = 
   mutableMapOf<Int, BigInteger>()

fun fib(n: Int): BigInteger = FIB_CACHE.getOrPut(n) {
   if (n <= 1) BigInteger.ONE else fib(n - 1) + fib(n - 2)
}
```

Now our method during the first run is nearly as efficient as a linear solution, and later it gives result immediately if it was already calculated. Comparison between this and classic linear fibonacci implementation on an example machine is presented in the below table. Also the iterative implementation we compare it to is presented below. 

|             | n = 100 | n = 200 | n = 300  | n = 400  |
| :---------- | :------ | :------ | :------- | :------- |
| fibIter     | 1997 ns | 5234 ns | 7008 ns  | 9727 ns  |
| fib (first) | 4413 ns | 9815 ns | 15484 ns | 22205 ns |
| fib (later) | 8 ns    | 8 ns    | 8 ns     | 8 ns     |

``` kotlin
fun fibIter(n: Int): BigInteger {
   if(n <= 1) return BigInteger.ONE
   var p = BigInteger.ONE
   var pp = BigInteger.ONE
   for (i in 2..n) {
       val temp = p + pp
       pp = p
       p = temp
   }
   return p
}
```

You can see that using this function for the first time is slower than using the classic approach as there is additional overhead on checking if the value is in the cache and adding it there. Once values are added, the retrieval is nearly instantaneous. 

It has a significant drawback though: we are reserving and using more memory since the `Map` needs to be stored somewhere. Everything would be fine if this was cleared at some point. But take into account that for the Garbage Collector (GC), there is no difference between a cache and any other static field that might be necessary in the future. It will hold this data as long as possible, even if we never use the `fib` function again. One thing that helps is using a soft reference that can be removed by the GC when memory is needed. It should not be confused with weak reference. In simple words, the difference is:

- Weak references do not prevent Garbage Collector from cleaning-up the value. So once no other reference (variable) is using it, the value will be cleaned.
- Soft references are not guaranteeing that the value won’t be cleaned up by the GC either, but in most JVM implementations, this value won’t be cleaned unless memory is needed. Soft references are perfect when we implement a cache.

This is an example property delegate (details in *Item 21: Use property delegation to extract common property patterns*) that on-demand creates a map and lets us use it, but does not stop Garbage Collector from recycling this map when memory is needed (full implementation should include thread synchronization):

``` kotlin
private val FIB_CACHE: MutableMap<Int, BigInteger> by
 SoftReferenceDelegate { mutableMapOf<Int, BigInteger>() }

fun fib(n: Int): BigInteger = FIB_CACHE.getOrPut(n) {
   if (n <= 1) BigInteger.ONE else fib(n - 1) + fib(n - 2)
}

class SoftReferenceDelegate<T: Any>(
   val initialization: ()->T
) {
   private var reference: SoftReference<T>? = null

   operator fun getValue(
       thisRef: Any?,
       property: KProperty<*>
   ): T {
       val stored = reference?.get()
       if (stored != null) return stored
       val new = initialization()
       reference = SoftReference(new)
       return new
   }
}
```

Designing a cache well is not easy, and in the end, caching is always a tradeoff: performance for memory. Remember this, and use caches wisely. No-one wants to move from performance issues to lack of memory issues. 

### Heavy object lifting

A very useful trick for performance is lifting a heavy object to an outer scope. Surely, we should lift if possible all heavy operations from collection processing functions to a general processing scope. For instance, in this function we can see that we will need to find the maximal element for every element in the `Iterable`:

``` kotlin
fun <T: Comparable<T>> Iterable<T>.countMax(): Int = 
    count { it == this.max() }
```

A better solution is to extract the maximal element to the level of `countMax` function:

``` kotlin
fun <T: Comparable<T>> Iterable<T>.countMax(): Int {
   val max = this.max()
   return count { it == max }
}
```

This solution is better for performance because we will not need to find the maximal element on the receiver on every iteration. Notice that it also improves readability by making it visible that `max` is called on the extension receiver and so it is the same through all iterations. 

Extracting a value calculation to an outer scope to not calculate it is an important practice. It might sound obvious, but it is not always so clear. Just take a look at this function where we use a regex to validate if a string contains a valid IP address:

``` kotlin
fun String.isValidIpAddress(): Boolean {
   return this.matches("\\A(?:(?:25[0-5]|2[0-4][0-9]
|[01]?[0-9][0-9]?)\\.){3}(?:25[0-5]|2[0-4][0-9]|[01]
?[0-9][0-9]?)\\z".toRegex())
}

// Usage
print("5.173.80.254".isValidIpAddress()) // true
```

The problem with this function is that `Regex` object needs to be created every time we use it. It is a serious disadvantage since regex pattern compilation is a complex operation. This is why this function is not suitable to be used repeatedly in performance-constrained parts of our code. Though we can improve it by lifting regex up to the top-level:

``` kotlin
private val IS_VALID_EMAIL_REGEX = "\\A(?:(?:25[0-5]
|2[0-4][0-9]|[01]?[0-9][0-9]?)\\.){3}(?:25[0-5]|2[0-4]
[0-9]|[01]?[0-9][0-9]?)\\z".toRegex()

fun String.isValidIpAddress(): Boolean = 
    matches(IS_VALID_EMAIL_REGEX)
```

If this function is in a file together with some other functions and we don’t want to create this object if it is not used, we can even initialize the regex lazily:

``` kotlin
private val IS_VALID_EMAIL_REGEX by lazy { 
"\\A(?:(?:25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\\.)
{3}(?:25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\\z".toRegex() 
}
```

Making properties lazy is also useful when we are dealing with classes. 

### Lazy initialization

Often when we need to create a heavy class, it is better to do that lazily. For instance, imagine that class `A` needs instances of `B`, `C`, and `D` that are heavy. If we just create them during class creation, `A` creation will be very heavy, because it will need to create `B`, `C` and `D` and then the rest of its body. The heaviness of objects creation will just accumulates.

``` kotlin
class A {
   val b = B()
   val c = C()
   val d = D()

   //...
}
```

There is a cure though. We can just initialize those heavy objects lazily:

``` kotlin
class A {
   val b by lazy { B() }
   val c by lazy { C() }
   val d by lazy { D() }

   //...
}
```

Each object will then be initialized just before its first usage. The cost of those objects creation will be spread instead of accumulated. 

Keep in mind that this sword is double-edged. You might have a case where object creation can be heavy, but you need methods to be as fast as possible. Imagine that `A` is a controller in a backend application that responds to HTTP calls. It starts quickly, but the first call requires all heavy objects initialization. So the first call needs to wait significantly longer for the response, and it doesn’t matter for how long our application runs. This is not the desired behavior. This is also something that might clutter our performance tests. 

### Using primitives

In JVM we have a special built-in type to represent basic elements like number or character. They are called primitives, and they are used by Kotlin/JVM compiler under the hood wherever possible. Although there are some cases where a wrapped class needs to be used instead. The two main cases are:

1. When we operate on a nullable type (primitives cannot be `null`)
2. When we use the type as a generic

So in short:

| Kotlin type | Java type     |
| :---------- | :------------ |
| Int         | int           |
| Int?        | Integer       |
| List<Int>   | List<Integer> |

Knowing that, you can optimize your code to have primitives under the hood instead of wrapped types. Such optimization makes sense mainly on Kotlin/JVM and on some flavours of Kotlin/Native. Not at all on Kotlin/JS. It also needs to be remembered that it makes sense only when operations on a number are repeated many times. Access of both primitive and wrapped types are relatively fast compared to other operations. The difference manifests itself when we deal with a really big collections (we will discuss it in the *Item 51: Consider Arrays with primitives for performance critical processing*) or when we operate on an object intensively. Also remember that forced changes might lead to less readable code. **This is why I suggest this optimization only for performance critical parts of our code and in libraries.** You can find out what is performance critical using a profiler.

To see an example, imagine that you implement a standard library for Kotlin, and you want to introduce a function that will return the maximal element or `null` if this iterable is empty. You don’t want to iterate over the iterable more than once. This is not a trivial problem, but it can be solved with the following function:

``` kotlin
fun Iterable<Int>.maxOrNull(): Int? {
   var max: Int? = null
   for (i in this) {
       max = if(i > (max ?: Int.MIN_VALUE)) i else max
   }
   return max
}
```

This implementation has serious disadvantages:

1. We need to use an Elvis operator in every step
2. We use a nullable value, so under the hood in JVM, there will be an `Integer` instead of an `int`. 

Resolving these two problems requires us to implement the iteration using while loop:

``` kotlin
fun Iterable<Int>.maxOrNull(): Int? {
   val iterator = iterator()
   if (!iterator.hasNext()) return null
   var max: Int = iterator.next()
   while (iterator.hasNext()) {
       val e = iterator.next()
       if (max < e) max = e
   }
   return max
}
```

For a collection of elements from 1 to 10 million, in my computer, the optimized implementation took 289 ms, while the previous one took 518 ms. This is nearly 2 times faster, but remember that this is an extreme case designed to show the difference. Such optimization is rarely reasonable in a code that is not performance-critical. Though if you implement a Kotlin standard library, everything is performance critical. This is why the second approach is chosen here:

``` kotlin
/**
* Returns the largest element or `null` if there are
* no elements.
*/
public fun <T : Comparable<T>> Iterable<T>.max(): T? {
   val iterator = iterator()
   if (!iterator.hasNext()) return null
   var max = iterator.next()
   while (iterator.hasNext()) {
       val e = iterator.next()
       if (max < e) max = e
   }
   return max
}
```

### Summary

In this item, you’ve seen different ways to avoid object creation. Some of them are cheap in terms of readability: those should be used freely. For instance, heavy object lifting out of a loop or function is generally a good idea. It is a good practice for performance, and also thanks to this extraction, we can name this object so we make our function easier to read. Those optimizations that are tougher or require bigger changes should possibly be skipped. We should avoid premature optimization unless we have such guidelines in our project or we develop a library that might be used in who-knows-what-way. We’ve also learned some optimizations that might be used to optimize the performance critical parts of our code.