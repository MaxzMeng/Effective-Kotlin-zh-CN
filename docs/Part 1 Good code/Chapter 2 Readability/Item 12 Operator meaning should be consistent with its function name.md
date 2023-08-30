## 第12条：操作符的含义应与其函数名一致

操作符重载是一项功能强大的特性，但与大多数强大功能一样，它也是危险的。在编程中，伴随强大的能力而来的是巨大的责任。作为一名培训师，我经常看到人们在刚刚发现操作符重载时会过度使用。例如，一个练习涉及创建一个计算阶乘的函数：

``` kotlin
fun Int.factorial(): Int = (1..this).product()

fun Iterable<Int>.product(): Int = 
    fold(1) { acc, i -> acc * i }
```

由于这个函数被定义为`Int`的扩展函数，使用起来非常方便：

``` kotlin
print(10 * 6.factorial()) // 7200
```

数学家知道，阶乘有一个特殊的表示法，即在数字后面加一个感叹号：

``` kotlin
10 * 6!
```

在Kotlin中并没有支持这样的操作符，但是正如我的一个研讨会参与者注意到的那样，我们可以使用`not`进行操作符重载：

``` kotlin
operator fun Int.not() = factorial()

print(10 * !6) // 7200
```

虽然我们可以这样做，但我们是否应该这样做呢？最简单的答案是不应该。只需要读取函数的声明就可以注意到，这个函数的名称是`not`。正如这个名称所暗示的，它不应该被这样使用。它表示的是逻辑操作，而不是数值阶乘。这种用法会导致混乱和误导。在Kotlin中，所有操作符只是具有具体名称的函数的语法糖。下表展示了每个操作符在Kotlin中对应的具体名称。每个操作符都可以以函数的形式调用，而不是使用操作符的语法。下面的代码会是什么样子呢？

``` kotlin
print(10 * 6.not()) // 7200
```

![What each operator translates to in Kotlin.](../../assets/chapter2/chapter2-2.png)

在Kotlin中，每个操作符的含义始终保持不变。这是一个非常重要的设计决策。有些语言，比如Scala，允许您无限制地进行操作符重载。这种自由度被一些开发人员滥用的现象广为人知。即使函数和类的名称有意义，第一次阅读使用不熟悉的库的代码可能也很困难。现在想象一下，操作符被用于另一种意义，只有熟悉*范畴论*的开发人员才知道。理解起来将更加困难。您需要分别理解每个操作符，在特定上下文中记住它的含义，然后将所有这些都记住以连接各个部分来理解整个语句。在Kotlin中，我们没有这样的问题，因为每个操作符都有一个具体的含义。例如，当您看到以下表达式时：

``` kotlin
x + y == z
```

您知道这等同于：

``` kotlin
x.plus(y).equal(z)
```

或者如果`plus`声明了可为空的返回类型，可以是以下代码：

``` kotlin
(x.plus(y))?.equal(z) ?: (z === null)
```

这些都是具有具体名称的函数，我们期望所有函数都按照它们的名称所指示的方式工作。这严格限制了每个操作符可以用于什么目的。使用`not`来返回`factorial`是对这个约定的明显违反，不应该发生。

### 不清晰的情况

最大的问题是在某些情况下不清楚某种用法是否符合约定。例如，当我们将一个函数三倍化时，这意味着什么？对于某些人来说，意思是创建另一个重复该函数三次的函数：

``` kotlin
operator fun Int.times(operation: () -> Unit): ()->Unit = 
    { repeat(this) { operation() } }

val tripledHello = 3 * { print("Hello") }

tripledHello() // Prints: HelloHelloHello
```

对于其他人来说，意思是我们要调用该函数三次：

``` kotlin
operator fun Int.times(operation: ()->Unit) {
   repeat(this) { operation() }
}

3 * { print("Hello") } // Prints: HelloHelloHello
```

当意义不清楚时，最好使用描述性的扩展函数。如果我们希望保持类似操作符的语法，可以使用`infix`修饰符或顶层函数：

``` kotlin
infix fun Int.timesRepeated(operation: ()->Unit) = {
   repeat(this) { operation() }
}

val tripledHello = 3 timesRepeated { print("Hello") }
tripledHello() // Prints: HelloHelloHello
```

有时候最好使用一个顶层函数。重复调用函数三次的功能已经在标准库中实现：

``` kotlin
repeat(3) { print("Hello") } // Prints: HelloHelloHello
```

### 什么时候可以打破这个规则？

有一个非常重要的情况可以打破这个规则：当我们设计领域特定语言（DSL）时。想象一个经典的HTML DSL示例：

``` kotlin
body {
    div {
        +"Some text"
    }
}
```

您可以看到，为了将文本添加到元素中，我们使用了`String.unaryPlus`。这是可以接受的，因为它显然是领域特定语言（DSL）的一部分。在这个特定的上下文中，读者并不会对使用不同规则感到惊讶。

### 总结

要谨慎使用操作符重载。函数名称应与其行为一致。避免操作符含义不清楚的情况。通过使用具有描述性名称的常规函数来澄清。如果希望具有更类似操作符的语法，则可以使用`infix`修饰符或顶层函数。