## 第9条：使用 `use`关闭资源

有些资源不能自动关闭，当我们不再使用它们时，我们需要调用`close`方法来手动关闭。我们在Kotlin/JVM中使用的Java标准库包含了很多这样的资源，例如

- `InputStream` 和`OutputStream`
- `java.sql.Connection`,
- `java.io.Reader` （`FileReader`, `BufferedReader`, `CSSParser`）
- `java.new.Socket` 和`java.util.Scanner`

所有这些资源都实现了`Closeable `接口，它继承自`AutoCloseable `。

对于上面列举的这些例子，当我们确认不再需要该资源的时候，我们需要调用`close`方法，因为这些资源的调用开销比较大并且它们被自动关闭的成本也较高（如果没有任何对该资源的引用，垃圾收集器最终会将它关闭，但这一过程所需的时间会比较久）。 因此，为了确保我们不会漏掉关闭它们，我们通常将这些资源调用放在在一个`try-finally`块中，并在`finally`中调用`close`方法：

``` kotlin
fun countCharactersInFile(path: String): Int {
   val reader = BufferedReader(FileReader(path))
   try {
       return reader.lineSequence().sumBy { it.length }
   } finally {
       reader.close()
   }
}
```

这样的结构是复杂且不正确的。它的不正确体现在`close`可能会抛出异常且这个异常不会被捕获。此外，如果我们同时从`try`和`finally`块中抛出异常，那么只有一个异常会被正确地传递。 我们所期望的表现应该是后抛出的异常信息应该被添加到之前已经抛出的异常信息中。正确的实现很长并且很复杂，但这种处理很常见，因此Kotlin标准库中提供了`use`函数。应该使用它来正确关闭资源和处理异常，此函数可用于任何`Closeable `对象：

``` kotlin
fun countCharactersInFile(path: String): Int {
   val reader = BufferedReader(FileReader(path))
   reader.use {
       return reader.lineSequence().sumBy { it.length }
   }
}
```

调用`use`的对象(本例中为`reader`)也会作为参数传递给lambda表达式，因此语法可以缩短:

``` kotlin
fun countCharactersInFile(path: String): Int {
   BufferedReader(FileReader(path)).use { reader ->
       return reader.lineSequence().sumBy { it.length }
   }
}
```

因为`use`函数经常被用来操作文件，同时逐行读取文件也是一种很常见的操作，所以Kotlin标准库中提供了一个` useLines`函数，它会返回给我们一个包含了文件中每一行内容（`String`类型）的序列，并且在读取完毕之后会自动关闭文件资源：

``` kotlin
fun countCharactersInFile(path: String): Int {
   File(path).useLines { lines ->
       return lines.sumBy { it.length }
   }
}
```

这是一种适合用来处理大文件的方法，因为序列会按需读取每一行，因此每次调用对于内存的占用不会超过一行的内容所对应的内存大小。 但代价是这个序列只能使用一次，如果你需要多次遍历文件，则需要多次调用它。 `useLines` 函数同样也能作为一个表达式来调用。

``` kotlin
fun countCharactersInFile(path: String): Int =
   File(path).useLines { lines -> 
       lines.sumBy { it.length } 
   }
```

以上所有使用序列对文件进行操作的例子，都是比较合理的处理方法。因为这样我们可以每次只加载一行的内容，避免直接加载整个文件。更多相关内容请参考 *Item 49: Prefer Sequence for big collections with more than one processing step*.

### Summary

使用`use`对实现了`Closeable`或`AutoCloseable`的对象进行操作，是一个安全且简单的选择。当你需要操作一个文件时，考虑使用`useLines`，它会生成一个序列来帮助你遍历每一行。