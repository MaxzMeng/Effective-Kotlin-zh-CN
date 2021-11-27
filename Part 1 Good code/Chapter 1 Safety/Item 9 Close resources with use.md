## Item 9: Close resources with `use`

There are resources that cannot be closed automatically, and we need to invoke the `close` method once we do not need them anymore. The Java standard library, that we use in Kotlin/JVM, contains a lot of these resources, such as:

- `InputStream` and `OutputStream`,
- `java.sql.Connection`,
- `java.io.Reader` (`FileReader`, `BufferedReader`, `CSSParser`),
- `java.new.Socket` and `java.util.Scanner`.

All these resources implement the `Closeable` interface, which extends `AutoCloseable`. 

The problem is that in all these cases, we need to be sure that we invoke the `close` method when we no longer need the resource because these resources are rather expensive and they aren’t easily closed by themselves (the Garbage Collector will eventually handle it if we do not keep any reference to this resource, but it will take some time). Therefore, to be sure that we will not miss closing them, we traditionally wrapped such resources in a `try-finally` block and called `close` there:

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

Such a structure is complicated and incorrect. It is incorrect because close can throw an error, and such an error will not be caught. Also, if we had errors from both the body of the `try` and from `finally` blocks, only one would be properly propagated. The behavior we should expect is for the information about the new error to be added to the previous one. The proper implementation of this is long and complicated, but it is also common and so it has been extracted into the `use`function from the standard library. It should be used to properly close resources and handle exceptions. This function can be used on any `Closeable` object:

``` kotlin
fun countCharactersInFile(path: String): Int {
   val reader = BufferedReader(FileReader(path))
   reader.use {
       return reader.lineSequence().sumBy { it.length }
   }
}
```

Receiver (`reader` in this case) is also passed as an argument to the lambda, so the syntax can be shortened:

``` kotlin
fun countCharactersInFile(path: String): Int {
   BufferedReader(FileReader(path)).use { reader ->
       return reader.lineSequence().sumBy { it.length }
   }
}
```

As this support is often needed for files, and as it is common to read files line-by-line, there is also a `useLines` function in the Kotlin Standard Library that gives us a sequence of lines (`String`) and closes the underlying reader once the processing is complete:

``` kotlin
fun countCharactersInFile(path: String): Int {
   File(path).useLines { lines ->
       return lines.sumBy { it.length }
   }
}
```

This is a proper way to process even large files as this sequence will read lines on-demand and does not hold more than one line at a time in memory. The cost is that this sequence can be used only once. If you need to iterate over the lines of the file more than once, you need to open it more than once. The `useLines` function can be also used as an expression:

``` kotlin
fun countCharactersInFile(path: String): Int =
   File(path).useLines { lines -> 
       lines.sumBy { it.length } 
   }
```

All the above implementations use sequences to operate on the file and it is the correct way to do it. Thanks to that we can always read only one line instead of loading the content of the whole file. More about it in the item *Item 49: Prefer Sequence for big collections with more than one processing step*.

### Summary

Operate on objects implementing `Closeable` or `AutoCloseable` using `use`. It is a safe and easy option. When you need to operate on a file, consider `useLines` that produces a sequence to iterate over the next lines.