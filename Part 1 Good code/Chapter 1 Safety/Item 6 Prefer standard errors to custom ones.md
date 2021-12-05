## 第6条：尽可能使用标准库中提供的异常

`require`, `check` 和`assert`函数涵盖了Kotlin中最常见的异常情况， 但还是有很多其他情况需要我们去主动声明。例如，当你实现一个库来解析JSON时，当提供的JSON文件格式不正确时时，抛出一个`JsonParsingException`是合理的：

``` kotlin
inline fun <reified T> String.readObject(): T {
   //...
   if (incorrectSign) {
       throw JsonParsingException()
   }
   //...
   return result
}
```

这里我们使用了一个自定义异常，因为在标准库中没有合适的异常来表答这种情况。你应该尽可能使用标准库提供的异常，而不是定义自己定义的异常。这些标准库提供的异常应该被开发者熟知和复用。在特定场景下使用这些异常会使你的API更容易学习和理解。下面是一些你可以使用的最常见的异常：

- `IllegalArgumentException` 和`IllegalStateException` - 像在第5条中提到的那样，使用 `require` 和`check` 来抛出该异常。
- `IndexOutOfBoundsException` - 表示索引越界。 常用于集合和数组。比如在 `ArrayList.get(Int)`中就会抛出该异常。
- `ConcurrentModificationException` - 表示并发修改是禁止的，当检测到这种行为时会抛出该异常。
- `UnsupportedOperationException` - 表示该对象不支持它声明的方法。我们应该避免这种情况，当一个方法不受支持时，它就不应该被声明在类中。
- `NoSuchElementException` - 表示被请求的元素不存在。 例如，当我们实现`Iterable`时，迭代器内已经没有其他元素了但是还是调用了 `next` 方法。