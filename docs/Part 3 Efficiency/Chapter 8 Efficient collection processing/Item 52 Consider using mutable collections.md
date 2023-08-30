## 第52条：在处理局部变量时，考虑使用可变集合

使用可变集合而不是不可变集合的最大优势是，它们在性能上表现更好。当我们向一个不可变的集合添加一个元素时，我们需要创建一个新的集合并将所有的元素添加到其中。下面是目前在Kotlin stdlib（Kotlin 1.2）中的实现方式：

``` kotlin
operator fun <T> Iterable<T>.plus(element: T): List<T> {
   if (this is Collection) return this.plus(element)
   val result = ArrayList<T>()
   result.addAll(this)
   result.add(element)
   return result
}
```

当我们处理数据量较大的集合时，添加前一个集合中的所有元素是一个巨大的性能开销。这就是为什么使用可变集合是一种性能优化，特别是当我们需要添加元素时。另一方面，*第1条：限制可变性*告诉我们使用不可变的集合来保证安全的好处。不过请注意，这些论点很少适用于局部变量，因为局部变量很少需要同步或封装。这就是为什么对于局部变量处理来说，通常使用可变集合更有意义。这一论点可以在标准库中得到证实，所有的集合处理函数在内部都是使用可变集合实现的：

``` kotlin
inline fun <T, R> Iterable<T>.map(
    transform: (T) -> R
): List<R> {
   val size = if (this is Collection<*>) this.size else 10
   val destination = ArrayList<R>(size)
   for (item in this)
       destination.add(transform(item))
   return destination
}
```

而不是使用不可变的集合：

``` kotlin
// This is not how map is implemented
inline fun <T, R> Iterable<T>.map(
    transform: (T) -> R
): List<R> {
   var destination = listOf<R>()
   for (item in this)
       destination += transform(item)
   return destination
}
```

### 总结

使用可变集合来添加元素通常有更好的性能表现，但是不可变集合给了我们更多的控制权来控制它们如何被更改。但是在局部变量范围内，我们通常不需要这种控制，所以应首选可变集合。特别是在工具类中，元素的插入可能会发生很多次。