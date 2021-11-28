## Item 51: Consider Arrays with primitives for performance-critical processing

We cannot declare primitives in Kotlin, but as an optimization, they are used under the hood. This is a significant optimization, described already in *Item 45: Avoid unnecessary object creation*. Primitives are:

- Lighter, as every object adds additional weight,
- Faster, as accessing the value through accessors is an additional cost.

Therefore using primitives for a huge amount of data might be a significant optimization. One problem is that in Kotlin typical collections, like `List` or `Set`, are generic types. Primitives cannot be used as generic types, and so we end up using wrapped types instead. This is a convenient solution that suits most cases, as it is easier to do processing over standard collections. Having said that, in the performance-critical parts of our code we should instead consider using arrays with primitives, like `IntArray` or `LongArray`, as they are lighter in terms of memory and their processing is more efficient. 

| Kotlin type | Java type     |
| :---------- | :------------ |
| Int         | int           |
| List<Int>   | List<Integer> |
| Array<Int>  | Integer[]     |
| IntArray    | int[]         |

How much lighter arrays with primitives are? Let’s say that in Kotlin/JVM we need to hold 1 000 000 integers, and we can either choose to keep them in `IntArray` or in `List<Int>`. When you make measurements, you will find out that the `IntArray` allocates 4 000 016 bytes, while `List<Int>` allocates 20 000 040 bytes. It is 5 times more. If you optimize for memory use and you keep collections of types with primitive analogs, like `Int`, choose arrays with primitives. 

``` kotlin
import jdk.nashorn.internal.ir.debug.ObjectSizeCalculator.getObjectSize

fun main() {
   val ints = List(1_000_000) { it }
   val array: Array<Int> = ints.toTypedArray()
   val intArray: IntArray = ints.toIntArray()
   println(getObjectSize(ints))     // 20 000 040
   println(getObjectSize(array))    // 20 000 016
   println(getObjectSize(intArray)) //  4 000 016
}
```

There is also a difference in performance. For the same collection of 1 000 000 numbers, when calculating an average, the processing over primitives is around 25% faster.

``` kotlin
open class InlineFilterBenchmark {

    lateinit var list: List<Int>
    lateinit var array: IntArray

    @Setup
    fun init() {
        list = List(1_000_000) { it }
        array = IntArray(1_000_000) { it }
    }

    @Benchmark
    // On average 1 260 593 ns
    fun averageOnIntList(): Double {
        return list.average()
    }

    @Benchmark
    // On average 868 509 ns
    fun averageOnIntArray(): Double {
        return array.average()
    }
}
```

As you can see, primitives and arrays with primitives can be used as an optimization in performance-critical parts of your code. They allocate less memory and their processing is faster. Although improvement in most cases is not significant enough to use arrays with primitives by default instead of lists. Lists are more intuitive and much more often in use, so in most cases we should use them instead. Just keep in mind this optimization in case you need to optimize some performance-critical parts.

### Summary

In a typical case, `List` or `Set` should be preferred over `Array`. Though if you hold big collections of primitives, using `Array` might significantly improve your performance and memory use. This item is especially for library creators or developers writing games or advanced graphic processing.