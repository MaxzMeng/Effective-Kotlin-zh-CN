## 第2条：最小化变量作用域

当我们定义一个状态时，我们倾向于通过以下方式收紧变量和属性的范围:

- 使用局部变量代替属性
- 在尽可能小的范围内使用变量，例如，如果一个变量只在循环中使用，那么就在这个循环中定义它

元素的作用域是指计算机程序中该元素可见的区域。在Kotlin中，作用域几乎总是由花括号创建的，我们通常可以从外部作用域访问元素。看看这个例子:

``` kotlin
val a = 1
fun fizz() {
   val b = 2
   print(a + b)
}
val buzz = {
   val c = 3
   print(a + c)
}
// 这里可以访问到 a ,但是无法访问 b 或 c
```

在上面的例子中，在函数` fizz `和` buzz `的作用域中，我们可以从函数作用域中访问外部作用域的变量。但是，在外部作用域中，我们不能访问这些函数中定义的变量。下面是一个限制变量作用域的示例:

``` kotlin
// 不好的写法
var user: User
for (i in users.indices) {
   user = users[i]
   print("User at $i is $user")
}

// 较好的写法
for (i in users.indices) {
   val user = users[i]
   print("User at $i is $user")
}

// 相同的变量作用域，更好的语法
for ((i, user) in users.withIndex()) {
   print("User at $i is $user")
}
```

在第一个例子中，` user ` 变量不仅在for循环的范围内可以访问，而且在for循环之外也可以访问。在第二个和第三个例子中，我们将变量` user ` 的作用域具体限制为for循环的作用域。

类似地，在作用域中可能还有许多作用域(最有可能的是嵌套在lambda表达式中的lambda表达式创建的)，最好在尽可能小的范围内定义变量。

我们这么做原因有很多，但最重要的是:**当我们收紧变量的作用域时，就会使我们的程序易于调试和管理。**当我们分析代码时，我们需要考虑此时存在哪些元素。需要处理的元素越多，编程就越困难。应用程序越简单，它崩溃的可能性就越小。这也是为什么我们更喜欢不可变属性或对象。

**考虑可变属性，当它们只能在较小的范围内修改时，更容易跟踪它们如何更改。**更容易对他们进行推理并改变他们的行为。

另一个问题是，**具有更大范围的变量可能会被其他开发人员过度使用**。例如，有人可能认为，如果使用一个变量来为迭代中的下一个元素赋值，那么在循环完成后，列表中的最后一个元素应该保留在该变量中。这样的推理可能导致严重的滥用，比如在迭代之后使用这个变量对最后一个元素做一些事情。这将是非常糟糕的，因为当另一个开发人员试图理解这个值的含义时，就需要理解整个执行过程。这将是一个不必要的麻烦。

**无论变量是只读的还是可读写的，我们总是倾向于在定义变量时就对其进行初始化。**不要强迫开发人员查看它的定义位置。这可以通过控制结构语句来实现，例如if, when, try-catch或Elvis操作符用作表达式：

``` kotlin
 // 不好的写法
val user: User
if (hasValue) {
   user = getValue()
} else {
   user = User()
}

// 较好的写法
val user: User = if(hasValue) {
   getValue()
} else {
   User()
}
```

如果我们需要设置多个属性，解构声明可以帮助我们更好的实现:

``` kotlin
// 不好的写法
fun updateWeather(degrees: Int) {
   val description: String
   val color: Int
   if (degrees < 5) {
       description = "cold"
       color = Color.BLUE
   } else if (degrees < 23) {
       description = "mild"
       color = Color.YELLOW
   } else {
       description = "hot"
       color = Color.RED
   }
   // ...
}

// 较好的写法
fun updateWeather(degrees: Int) {
   val (description, color) = when {
       degrees < 5 -> "cold" to Color.BLUE
       degrees < 23 -> "mild" to Color.YELLOW
       else -> "hot" to Color.RED
   }
   // ...
}
```

最后，太大的变量范围可能是危险的。让我们来看一个例子。

### 变量捕获

当我在教授 Kotlin 协程时，我布置的练习之一是使用序列构建器实现 Eratosthenes 算法以查找素数。 该算法在概念上很简单：

1. 创建一个从 2 开始的数字列表。
2. 取第一个数，它是一个素数。
3. 从其余的数字中，我们删除第一个数字，并过滤掉所有可以被这个素数整除的数字。

该算法的一个简单实现如下所示：

``` kotlin
var numbers = (2..100).toList()
val primes = mutableListOf<Int>()
while (numbers.isNotEmpty()) {
   val prime = numbers.first()
   primes.add(prime)
   numbers = numbers.filter { it % prime != 0 }
}
print(primes) // [2, 3, 5, 7, 11, 13, 17, 19, 23, 29, 31,
// 37, 41, 43, 47, 53, 59, 61, 67, 71, 73, 79, 83, 89, 97]
```

这个问题的挑战在于如何让它产生一个可能无限的质数序列。 如果您想挑战自己，请立即停止阅读并尝试实现它。

这是一种解决方法：

``` kotlin
val primes: Sequence<Int> = sequence {
   var numbers = generateSequence(2) { it + 1 }

   while (true) {
       val prime = numbers.first()
       yield(prime)
       numbers = numbers.drop(1)
           .filter { it % prime != 0 }
   }
}

print(primes.take(10).toList())
// [2, 3, 5, 7, 11, 13, 17, 19, 23, 29]
```

在几乎每一组中都有一个人试图“优化”它。他们提取```prime```作为可变变量，而不是在每个循环中都创建变量：

``` kotlin
val primes: Sequence<Int> = sequence {
   var numbers = generateSequence(2) { it + 1 }

   var prime: Int
   while (true) {
       prime = numbers.first()
       yield(prime)
       numbers = numbers.drop(1)
           .filter { it % prime != 0 }
   }
}
```

但是这会导致这个实现不再能得到正确的结果。以下是前10个数字:

``` kotlin
print(primes.take(10).toList())
// [2, 3, 5, 6, 7, 8, 9, 10, 11, 12]
```

现在你可以停下来去尝试解释为什么会出现这样的结果。

我们得到这样结果的原因是我们捕获了变量`prime`。因为我们使用的是序列，所以过滤是惰性完成的。在每一步中，我们不断地在添加过滤器。而在“优化”版本的代码中，我们总是只添加引用可变属性`prime`的过滤器。 因此，我们总是过滤` prime`的最后一个值用来过滤。 这就是为什么我们不能过滤出正确的结果。只有drop函数生效了，所以我们得到的是一个连续的数字序列 (除了`prime `被设置为2时被过滤掉的4).

我们应该意识到这种无意捕获的问题，因为这种情况时有发生。为了防止这种情况，我们应该避免可变性，并使变量的作用域更小。

### 总结

出于许多原因，我们应该更倾向于在最小的范围内定义变量。同样，对于局部变量，我们更喜欢` val` 而不是`var `。我们应该始终意识到变量是在lambdas表达式中被捕获的。这些简单的规则可以为我们省去许多麻烦。
