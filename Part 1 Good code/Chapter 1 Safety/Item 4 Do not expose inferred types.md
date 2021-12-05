## 第4条：不要暴露推断类型

Kotlin的类型推断是JVM中最流行的Kotlin特性之一。以至于Java 10也引入了类型推断(与Kotlin相比有限)。 不过，使用该特性也有一些危险性。最重要的是，我们需要记住，赋值的推断类型是右边的确切类型，而不是超类或接口类型:

```kotlin
open class Animal
class Zebra: Animal()

fun main() {
   var animal = Zebra()
   animal = Animal() // Error: Type mismatch
}
```

在大多数情况下，这都不是一个难题。当我们需要限定推断出的类型时，我们只需要指定它，问题就解决了:

``` kotlin
open class Animal
class Zebra: Animal()

fun main() {
   var animal: Animal = Zebra()
   animal = Animal()
}
```

然而，当我们使用三方编写的一个库或另一个模块时，就没有这么容易了。在这种情况下，推断类型说明可能非常危险。让我们看一个例子。

假设你有以下接口用来代表汽车工厂:

``` kotlin
1 interface CarFactory {
2    fun produce(): Car
3 }
```

如果没有指定其他参数，也会使用默认的类型`Fiat126P`：

``` kotlin
1 val DEFAULT_CAR: Car = Fiat126P()
```

因为绝大多数汽车工厂都可以生产它，所以你把它设为默认值。你没有给它声明返回值类型，因为你认为`DEFAULT_CAR `无论如何都是`Car`的实例：

``` kotlin
1 interface CarFactory {
2    fun produce() = DEFAULT_CAR
3 } 
```

类似地, 后来有其他人看到 `DEFAULT_CAR`的声明并且认为它的类型能够被推断出来：

``` kotlin
1 val DEFAULT_CAR = Fiat126P()
```

现在你会发现所有的工厂都只能生产`Fiat126P`。这显然是有问题的。如果这个接口是你自己定义的，那么这个问题可能很快就会被发现并且很容易修复。但是，如果它作为外部API被提供给用户使用，你可能会从愤怒的用户那里得知这个问题。

除此之外，当某人不太了解API时，返回类型是很重要的信息。因此为了可读性，我们应该显式声明返回类型，特别是在我们的API的外部可见部分(即公开的API)中。

#### 总结

一般的规则是，如果我们不确定返回值类型，我们应该显示声明它。这是很重要的信息，我们不应该把它隐藏起来 (*Item 14: Specify the variable type when it is not clear*)。此外，为了安全起见，在外部API中，我们应该始终指定类型。不能让它们随意改变。当我们的项目迭代时，推断类型可能会有很多限制或者很容易被更改。