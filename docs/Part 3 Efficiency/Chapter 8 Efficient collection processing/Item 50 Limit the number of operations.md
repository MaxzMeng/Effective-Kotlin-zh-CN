## 第50条: 减少操作的次数

每个集合处理的方法都伴随着性能开销。对于标准库中提供的集合处理方法，在底层实现上，通常是对集合元素的一次迭代和新集合的创建。而对于序列处理，它会创建一个新的对象来持有和操作整个序列。尤其是当处理的元素数量很多的时候，这两者的性能开销都会变的很大。因此，我们应该减少处理步骤的数量，主要可以通过使用复合操作来做到这一点。例如，我们使用 `filterNotNull`，而不是先过滤非空类型然后再转换为不可空类型。或者我们可以只使用`mapNotNull`而不是先进行映射之后再进行过滤操作。

``` kotlin
class Student(val name: String?)

// Works
fun List<Student>.getNames(): List<String> = this
   .map { it.name }
   .filter { it != null }
   .map { it!! }

// Better
fun List<Student>.getNames(): List<String> = this
   .map { it.name }
   .filterNotNull()

// Best
fun List<Student>.getNames(): List<String> = this
   .mapNotNull { it.name }
```

通常来说，最大的问题不是缺乏对这一问题的认知，而是缺乏对应该使用哪些集合处理函数的了解。 这是学习它们很好的另一个原因。 此外，IDE也会提供有用的警告，这些警告经常给我们更好的替代方案的建议。

![](../../assets/chapter8/chapter8-6.png)

尽管如此，了解如何替代复合操作还是很有必要的。 下面列出了一些常见的函数调用和限制操作数量的替代方法：

![](../../assets/chapter8/chapter8-7.png)

### 总结

大多数集合处理步骤需要迭代整个集合和额外的集合创建。 这个成本可以通过使用更合适的函数来限制。