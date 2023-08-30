## 第15条：考虑明确指定接收者

有一个常见的情况，我们可能会选择使用更长的结构来使某些内容明确，即当我们想要强调函数或属性是来自接收者而不是局部或顶层变量时。在最基本的情况下，这意味着引用与方法关联的类：

``` kotlin
class User: Person() {
   private var beersDrunk: Int = 0
  
   fun drinkBeers(num: Int) {
       // ...
       this.beersDrunk += num
       // ...
   }
}
```

类似地，我们可以明确引用扩展接收者（扩展方法中的`this`）以使其使用更明确。比较以下两种不使用明确接收者编写的快速排序实现：

``` kotlin
fun <T : Comparable<T>> List<T>.quickSort(): List<T> {
   if (size < 2) return this
   val pivot = first()
   val (smaller, bigger) = drop(1)
       .partition { it < pivot }
   return smaller.quickSort() + pivot + bigger.quickSort()
}
```

与使用明确接收者编写的实现相比：

``` kotlin
fun <T : Comparable<T>> List<T>.quickSort(): List<T> {
   if (this.size < 2) return this
   val pivot = this.first()
   val (smaller, bigger) = this.drop(1)
       .partition { it < pivot }
   return smaller.quickSort() + pivot + bigger.quickSort()
}
```

两个函数的使用方式是相同的：

``` kotlin
listOf(3, 2, 5, 1, 6).quickSort() // [1, 2, 3, 5, 6]
listOf("C", "D", "A", "B").quickSort() // [A, B, C, D]
```

### 多个接收者

在涉及多个接收者的范围内，明确指定接收者特别有帮助。我们在使用`apply`、`with`或`run`函数时经常会遇到这种情况。这种情况很危险，我们应该避免使用它们。使用显式接收者使用对象更安全。为了理解这个问题，看看下面的代码：

``` kotlin
class Node(val name: String) {

   fun makeChild(childName: String) =
       create("$name.$childName")
           .apply { print("Created ${name}") }

   fun create(name: String): Node? = Node(name)
} 

fun main() {
   val node = Node("parent")
   node.makeChild("child")
}
```

结果是什么？在阅读答案之前，请停下来花点时间自己思考一下。

预计的结果可能是"Created parent.child"，但实际结果是"Created parent"。为什么？为了调查，我们可以在`name`之前使用明确的接收者：

``` kotlin
class Node(val name: String) {

   fun makeChild(childName: String) =
       create("$name.$childName")
         .apply { print("Created ${this.name}") } 
         // Compilation error

   fun create(name: String): Node? = Node(name)
}         
```

问题在于`apply`内部的`this`类型是`Node?`，所以不能直接使用方法。我们需要首先解包它，例如通过使用安全调用。如果我们这样做，结果最终将是正确的：

``` kotlin
class Node(val name: String) {

   fun makeChild(childName: String) =
       create("$name.$childName")
           .apply { print("Created ${this?.name}") }

   fun create(name: String): Node? = Node(name)
}

fun main() {
    val node = Node("parent")
    node.makeChild("child") 
    // Prints: Created parent.child
}
```

这是`apply`的错误使用示例。如果我们使用`also`代替，并在参数上调用`name`，就不会遇到这样的问题。使用`also`强制我们以明确的方式引用函数的接收者，就像使用明确接收者一样。通常情况下，`also`和`let`在执行额外操作或操作可为空的值时是更好的选择。

``` kotlin
class Node(val name: String) {

   fun makeChild(childName: String) =
       create("$name.$childName")
           .also { print("Created ${it?.name}") }

   fun create(name: String): Node? = Node(name)
}
```

当接收者不明确时，我们要么避免使用它，要么使用明确的接收者来澄清。当我们在没有标签的情况下使用接收者时，我们指的是最接近的接收者。当我们想要使用外部接收者时，需要使用标签。在这种情况下，明确使用它特别有用。以下是一个同时使用它们的示例：

``` kotlin
class Node(val name: String) {

    fun makeChild(childName: String) =
        create("$name.$childName").apply { 
           print("Created ${this?.name} in "+
               " ${this@Node.name}") 
        }

    fun create(name: String): Node? = Node(name)
}

fun main() {
    val node = Node("parent")
    node.makeChild("child") 
    // Created parent.child in parent
}
```

这样，直接接收者澄清了我们指的是哪个接收者。这可能是一个重要的信息，不仅可以保护我们免受错误的影响，还可以提高可读性。

### DSL标记

在某些情况下，我们经常在具有不同接收者的嵌套范围内操作，并且根本不需要使用显式接收者。我指的是Kotlin的DSL。我们不需要显式使用接收者，因为DSL是以这种方式设计的。然而，在DSL中，意外使用外部作用域的函数尤其危险。想象一下我们用于制作HTML表格的简单HTML DSL：

``` kotlin
table {
   tr {
       td { +"Column 1" }
       td { +"Column 2" }
   }
   tr {
       td { +"Value 1" }
       td { +"Value 2" }
   }
}
```

请注意，每个作用域默认情况下我们还可以使用外部作用域的接收者的方法。我们可以利用这一点来混淆DSL：

``` kotlin
table {
   tr {
       td { +"Column 1" }
       td { +"Column 2" }
       tr {
           td { +"Value 1" }
           td { +"Value 2" }
      }
   }
}
```

为了限制这种用法，我们有一个特殊的元注解（注解的注解），用于限制对外部接收者的隐式使用。这就是`DslMarker`。当我们在注解上使用它，并在稍后将此注解用于作为构建器的类时，在此构建器内部将无法隐式使用接收者。以下是`DslMarker`的用法示例：

``` kotlin
@DslMarker
annotation class HtmlDsl

fun table(f: TableDsl.() -> Unit) { /*...*/ }

@HtmlDsl
class TableDsl { /*...*/ }
```

通过这样做，禁止了对外部接收者的隐式使用：

``` kotlin
table {
   tr {
       td { +"Column 1" }
       td { +"Column 2" }
       tr { // COMPILATION ERROR
           td { +"Value 1" }
           td { +"Value 2" }
       }
   }
}
```

使用外部接收者的函数需要显式接收者的使用：

``` kotlin
table {
   tr {
       td { +"Column 1" }
       td { +"Column 2" }
       this@table.tr {
           td { +"Value 1" }
           td { +"Value 2" }
       }
   }
}
```

DSL标记是一种非常重要的机制，我们可以使用它来强制使用最近的接收者或显式接收者。然而，在DSL中，通常最好不要使用显式接收者。尊重DSL的设计，并相应地使用它。

### 总结

不要仅仅因为可以而更改作用域接收者。拥有太多的接收者并提供可以使用的方法可能会引起混淆。显式参数或引用通常更好。当我们确实更改接收者时，使用显式接收者可以提高可读性，因为它澄清了函数来自哪里。如果有多个接收者，甚至可以使用标签来澄清函数来自哪个接收者。如果您想要强制从外部作用域使用显式接收者，可以使用`DslMarker`元注解。