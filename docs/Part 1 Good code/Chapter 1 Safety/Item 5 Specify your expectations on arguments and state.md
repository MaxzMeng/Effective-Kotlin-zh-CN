## 第 5 项: 在参数与状态上指定你的期望

当你对引数(参数)有所期待时，尽快的声明它们，在Kotlin中我们大部分使用：

- `require` - 一种具体指定参数(arguments)期望的通用方法
- `check` - 一种具体指定状态(state)期望的通用方法
- `assert` - 检查某件事情是否属实的通用方法，在JVM上，assert只有在测试模式(debug mode)下使用
- `?:(Elvis operator)` -  带有return或throw的运算子

这里有个使用上述技巧的范例：

``` kotlin
// Part of Stack<T>
fun pop(num: Int = 1): List<T> {
   require(num <= size) { 
       "Cannot remove more elements than current size"
   }
   check(isOpen) { "Cannot pop from closed stack" }
   val ret = collection.take(num)
   collection = collection.drop(num)
   assert(ret.size == num)
   return ret
}
```

像上述例子，这种具体指定的方法并不能让我们免于在文档中指定这些期望，但无论如何，它确实很有帮助，这种声明式检查有很多优点：

- 即使那些没有阅读文档的开发人员也能看到那些期望
- 若该期望条件没有被满足，那么该函数会抛出异常，而不是导致非预期的行为，在状态被修改之前抛出异常是很重要的，因此我们不会出现某些修改有生效，而其他部分没有生效的状况，这种情况是危险且难以掌控的，归功于这种断言检查，程式码中的错误更难被遗漏，我们的状态也更稳定
- 程式碼在一定程度上是自檢( self-checking)的，當這些條件在程式碼中被檢查過後，就不太需要進行單元測試了
- 上述的四种(`require`,`check`,`assert`,`?:`)检查都适用于智能转型(`smart-casting`)，多亏了它们，需要的转换(casting)更少了

我们来谈谈不同类型的检查还有为何我们需要它，从最受欢迎的一个开始：参数检查

### 参数

当你定义一个带有参数的函数时，对于那些无法使用型别系统表达的参数带有一些期望是很常见的事，我们来看看一些例子：

- 当你在计算一个数字的阶乘时，你可能会要求这个数字要是一个正整数
- 当你在寻找丛集时，你可能会要求这些点列表不得为空
- 当你要向使用者发送电子邮件时，你可能会要求该使用者拥有电子邮件，且该电子邮件地址也是正确的(假设使用者在使用此功能之前已经确认过电子邮件的正确性)

在Kotlin中用来说明那些需求，最普遍也最直接的方法就是使用require函数来检查这些需求，并在无法满足该条件时抛出异常

``` kotlin
fun factorial(n: Int): Long {
   require(n >= 0)
   return if (n <= 1) 1 else factorial(n - 1) * n
}

fun findClusters(points: List<Point>): List<Cluster> {
   require(points.isNotEmpty())
   //...
}

fun sendEmail(user: User, message: String) {
   requireNotNull(user.email)
   require(isValidEmail(user.email))
   //...
}
```

请注意，这些需求是非常高度可见的，因为他们是在函数的最开始地方声明的，这使得阅读这个函数的人更清楚明了(尽管如此，这些需求也应该在文档中说明，因为不是每个人都会阅读函数内的程式码)

那些期望是不能被忽略的，因为require函数的叙述不被满足时会抛出 `IllegalArgumentException`，当这样的一个程式码区块放在函数一开始的地方我们就会知道如果参数是不正确的，那么这个函数就会抛出错误而立即停止，而且使用者无法忽视它，这与那些可能传播到很远直到失败的潜在异常结果恰恰相反，该异常将会是很明显的，换句话说，当我们在函数一开始的地方正确的指定我们对该参数的期望时，那我们可以假设这些期望将会获得满足。

我们还可以在调用之后在lambda表达式中替这个异常指定一个惰性消息(lazy message)

``` kotlin
fun factorial(n: Int): Long {
   require(n >= 0) { "Cannot calculate factorial of $n " +
"because it is smaller than 0" }
   return if (n <= 1) 1 else factorial(n - 1) * n
}
```

当我们对参数指定需求时，会使用`require`

另一种常见的状况是当我们对当前的状态有期望时，在这种情况下，可以使用`check`函数来替代

### 状态(State)

我们只允许在具体条件下使用某些函数的情况是很常见的，来看看一些常见的例子：

- 有些函数可能需要先初始化一个物件
- 仅当使用者登录时才允许操作
- 函数可能需要打开一个物件

檢查狀態的期望是否得到滿足的標準方法是使用`check`功能

``` kotlin
fun speak(text: String) {
   check(isInitialized)
   //...
}

fun getUserInfo(): UserInfo {
   checkNotNull(token)
   //...
}

fun next(): T {
   check(isOpen)
   //...
}
```

- `check`运作方式与`require`很类似，差别是当`check`内所声明的期望没有被满足时会抛出一个` IllegalStateException`，它用来检查状态是否正确，可以使用lazy message(懒消息)来自定义异常讯息，当这个期望是影响整个函数时，我们会将它放在函数一开始的地方，一般来说会放在`require`之后，或者某些状态期望是局部的，可以在之后再使用`check`
- 我们使用这些检查，尤其是当我们怀疑使用者可能会违反我们的规约并且在错误的时间点调用函数，与其相信他们不会那么做，不如检查并抛出适当的异常。当我们不相信自己的实作(implementation)能正确的处理状态时，我们也可以使用`check`，除了上述例子外，当我们的主要目的是为了测试我们自己的实作(implementation)而进行检查时，我们还有另一个函数叫做*assert*

### 断言(Assertions)

当一个函数功能被正确的实现时，我们知道有些事是正确的，例如：当一个函数被要求返回10个元素时，我们可能会期望它要返回10个元素，这是我们期待的结果，但这并不意味着我们总是对的！我们都会犯错，也许我们在实作(现)中犯了错，也许某人更动了我们使用的功能，而使我们的函数不再正常的运作，也许我们的函数在重构之后就不再正常的运作，对于上述这些问题，最通用的解决方法是我们应该撰写单元测试(`unit test`)来检查我们的期望是否与现实相符合

``` kotlin
class StackTest {
  
   @Test
   fun `Stack pops correct number of elements`() {
       val stack = Stack(20) { it }
       val ret = stack.pop(10)
       assertEquals(10, ret.size)
   }
  
   //...
}
```

单元测试应该是我们用来检查实作(现)正确性最基础的方法，请注意上方例子，弹出list(列表)大小与所需大小匹配这一事实对于这个测试函数是相当普遍的，在每个调用`pop`附近增加这种检查是非常有用的，对这种单一用途只进行一次检查是相当幼稚的，因为可能存在一些边界条件，更好的做法是在函数中加入断言

``` kotlin
fun pop(num: Int = 1): List<T> {
   //...
   assert(ret.size == num)
   return ret
}
```

此类条件目前仅在Kotlin/JVM中启用，除非在JVM option使用`-ea`来启用，否则他们不会被检查，我们宁用将它们视为单元测试的一部份--它们检查我们的程式码是否有按预期工作，在预设状况下，它们不会在正式版本上抛出任何错误，它们预设只有在测试模式下才会启用，一般来说，这是我们所需要的，因为如果我们犯了错，我们宁可希望使用者不会发现，如果这是一个会导致严重错误与重大后果的错误，那请改用`check`，在函数中使用`assert`检查会比在单元测试中检查来得好有下列几项优势：

断言(assertions)使程式码能够自我检查(self-checking)进而提升测试效率
    - 检查每个实际用例而不是具体案例的期望
    - 我们可以使用它们在确切的执行点检查某些东西
    - 它可以使程式码提早失败(当有错误时)，使我们更接近问题核心，多亏了这一点，我们还可以轻松的找到非预期行为发生的地点与时间

请记住，除了使用它们外，我们仍然需要撰写单元测试，因为标准的应用程式载运行中，`assert`不会抛出异常

这样的断言在Python是很常见的，Java则没有那么多，但是在Kotlin你可以随意的使用它们可以帮助程式码变得更可靠。

### 可空性与智能转换

`require`与`check`都有Kotlin合约，该合约声明了在该函数返回时它(`require(叙述句)`/`check(叙述句)`)叙述句的这个检查结果为真

``` kotlin
public inline fun require(value: Boolean): Unit {
   contract {
       returns() implies value
   }
   require(value) { "Failed requirement." }
}
```

那些在`require()`或`check()`区块内检查过的所有内容，稍后将在同一个函数内被视为`true`，这适用于智能转换(`smart-casting`)，因为一旦我们检查了某事为真(true)，编译器就会如此对待它，看下方例子，我们需要`person`的`outfit`是`Dress`，之后，假设`outfit`属性是`final`的，那么它会被智能转换成`Dress`

``` kotlin
fun changeDress(person: Person) {
   require(person.outfit is Dress)
   val dress: Dress = person.outfit
   //...
}
```

这个特性(`smart-casting`)在我们用来检查某些事情是否为空时特别有用

``` kotlin
class Person(val email: String?)

fun sendEmail(person: Person, message: String) {
   require(person.email != null)
   val email: String = person.email
   //...
}
```

对于这类情况，Kotlin还有特殊函数：`requireNotNull`和`checkNotNull`，它们都具有智能转换(`smart-casting`)的能力，也可以当成表达式(`expressions`)来`解包(unpack)`它

``` kotlin
class Person(val email: String?)
fun validateEmail(email: String) { /*...*/ }

fun sendEmail(person: Person, text: String) {
   val email = requireNotNull(person.email)
   validateEmail(email)
   //...
}

fun sendEmail(person: Person, text: String) {
   requireNotNull(person.email)
   validateEmail(person.email)
   //...
}
```

对于可空性(nullability)，使用Elvis运算子也是很受欢迎的，它可以在运算子右边使用`throw`或`return`，这样写结构也具有很高的可读性，它提供我们在决定要达成什么样的行为时更多的灵活性，首先，我们可以轻易的透过`return`终止一个函数而不是抛出异常

``` kotlin
fun sendEmail(person: Person, text: String) {
   val email: String = person.email ?: return
   //...
}
```

如果我们需要在一个属性出现非预期的状态(这里使用`null`来说明)时执行复数个操作，我们可以透过把`return`或`throw`包装近*run*函数中来达成，如果我们需要纪录函数停止的原因，这样的能力是很有用处的

``` kotlin
fun sendEmail(person: Person, text: String) {
   val email: String = person.email ?: run {
       log("Email not sent, no email address")
       return
   }
   //...
}
```

带有`return`或`throw`的Elvis运算子是一种流行且惯用的方式，来指定在变数可空性的情况下应该发生什么，我们应该毫不犹豫的使用它，如果可以的话，在函数开始的地方使用这些检查使使它们清楚可见

### 总结

指定你的期望：

- 使它们更明显
- 保护应用程式的稳定性
- 保护程式码的正确性
- 智能转换(`smart-casting`)变数

为了达成上述所列，有四种主要机制如下：

- `require` - 一种具体指定参数(arguments)期望的通用方法
- `check` - 一种具体指定状态(state)期望的通用方法
- `assert` - 检查某件事情是否属实的通用方法，在JVM上，assert只有在测试模式(debug mode)下使用
- `?:(Elvis operator)` -  带有return或throw的运算子

你也可以使用error throw来抛出不同的错误
