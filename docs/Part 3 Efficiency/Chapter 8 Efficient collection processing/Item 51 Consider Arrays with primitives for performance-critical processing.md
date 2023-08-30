## 第51条：在“性能优先”的场景，使用基础类型数组

我们不能在Kotlin中声明基础类型，但是作为一种优化手段，在底层经常会使用它。这是一个重要的优化，在*第45项：避免不必要的对象创建*中已经讲述过。基础类型是：

- 更轻量级的，因为每个对象都会增加额外的内存开销
- 更快，因为通过getter方法访问值有额外的性能开销

因此，对量较大的数据使用基础类型可能是一个重要的优化。但有一个问题是，在Kotlin中，典型的集合，如`List`或`Set`，使用的是泛型类型。基础类型不能作为泛型参数使用，因此我们最终使用的是包装类型。这是一个方便的解决方案，适合大多数情况，因为在标准集合上更容易做处理操作。尽管如此，在代码中对性能有重要要求的部分，我们应该考虑使用基础类型数组，比如`IntArray`或`LongArray`，因为它们的内存占用更少，处理效率更高。

| Kotlin type | Java type     |
| :---------- | :------------ |
| Int         | int           |
| List<Int>   | List<Integer> |
| Array<Int>  | Integer[]     |
| IntArray    | int[]         |

基础类型数组有多轻量级呢？  假设在Kotlin/JVM中，我们需要保存1 000 000个整数，我们可以选择将它们保存在`IntArray`或`List<Int>`中。当你对他们的内存情况进行计算时，你会发现`IntArray`分配了4 000 016字节的内存，而`List<Int>`分配了20 000 040字节，足足5倍的差距。 如果你需要做内存优化，并且你的集合持有的数据类型是基础类型， 如`Int`，那么你可以尝试使用基础类型数组来优化。

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

在性能上同样也有差别，对于同样的1 000 000个数字的集合，在计算平均值时，对基础类型数组的处理要快25%左右。

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

正如你所看到的，使用基础类型和基础类型数组可以作为代码中优化性能的一种手段。它们占用的内存更少，处理速度也更快。虽然在大多数情况下，这种优化带来的效果并不明显，不足以让我们默认使用基础类型数组而不是`List`。`List`更直观，使用频率更高，所以在大多数情况下，我们应该使用它。但你仍需记住这种优化，以防你需要优化一些”性能优先“的部分。

### 总结

在一般情况下，应该优先使用`List`或`Set`而不是`Array`。但如果你需要保存大量的基础类型的数据，使用`Array`可能会显著提高你的性能和内存使用情况。这一条特别适用于开发基础库或编写游戏或高级图形处理的开发者。