## 第7条：当不能返回预期结果时，优先使用`null` o或`Failure` 作为返回值

有时，函数不能返回预期的结果。常见的例子有：

- 我们试图从服务器获取数据，但我们的网络连接有问题
- 我们试图获取符合某些条件的第一个元素，但在我们的集合中没有这样的元素
- 我们试图把文本解析为一个对象，但是文本的格式是错误的

有两种主要的办法来处理这种情况:

- 返回一个`null`或一个表示失败的密封类（通常命名为 `Failure`）
- 抛出一个异常

这两者之间有一个重要的区别。异常不应该被用作传递信息的标准方式。**所有异常都是指不正确的、特殊的情况，应按这种方式处理。我们应该只在特殊情况下使用异常**（来自Joshua Bloch的Effective Java）**。**这样做的主要原因是:

- 异常传递的方式对大多数程序员来说可读性较差，并且很容易在代码中漏掉。
- 在Kotlin中，所有异常都未经检查。用户不会被强迫甚至鼓励去处理它们。它们通常没有很好地被标记出来。当我们使用API时，它们实际上是不可见的。
- 因为异常是为异常环境设计的，所以JVM的实现者们没有任何动机让它们有和正常情况一样的执行速度。
- 将代码放在try-catch块中会抑制编译器可能执行的某些优化。

另一方面，`null`或`Failure`都是表示预期错误的最佳选择。它们是显式的、高效的，并且可以用惯用的方式处理。这就是为什么**当发生预期内的错误时，我们应该更倾向于返回`null`或`Failure `时，而当发生预期外的错误时，应该更倾向于抛出一个异常。**以下是一些例子：

``` kotlin
inline fun <reified T> String.readObjectOrNull(): T? {
   //...
   if (incorrectSign) {
       return null
   }
   //...
   return result
}

inline fun <reified T> String.readObject(): Result<T> {
   //...
   if (incorrectSign) {
       return Failure(JsonParsingException())
   }
   //...
   return Success(result)
}

sealed class Result<out T>
class Success<out T>(val result: T) : Result<T>()
class Failure(val throwable: Throwable) : Result<Nothing>()

class JsonParsingException : Exception()
```

当我们选择返回`null`时，我们可以从各种支持空安全的特性中选择来处理这样的值的方式，比如安全调用或使用Elvis操作符：

``` kotlin
val age = userText.readObjectOrNull<Person>()?.age ?: -1
```

当我们选择返回像`Result`这样的复合类型时，用户将能够使用When表达式来处理它:

``` kotlin
val personResult = userText.readObject<Person>()
val age = when(personResult) {
    is Success -> personResult.value.age
    is Failure -> -1
}
```

使用这种错误处理不仅比try-catch块更有效，而且通常更容易使用，可读性更好。一个异常可能会被错过，并可能导致整个应用程序停止。虽然`null`值或密封的结果类需要显式处理，但它不会中断应用程序的流程。

当在`null`和`Failure`两者间进行选择时，如果失败时需要传递额外的信息，我们应该选择后者，否则应该选择`null`。`Failure`可以保存你需要的任何数据。

通常来说我们应该对一个函数提供两种变体——一种会抛出异常，另一种不会。一个很好的例子是`List`有以下两个方法：

- `get` 返回给定下标位置的元素，如果对应的下标超出了列表范围， 这个方法会抛出`IndexOutOfBoundsException`.
- `getOrNull`，当我们知道我们可能会访问一个超出列表范围的元素时，我们应该使用它，这样当下标超出范围时，它会返回给我们 `null`. 

它还支持其他选项，如`getOrDefault`，这在某些情况下很有用，但通常可能很容易替换为`getOrNull`和Elvis操作符`?:`。

这是一个很好的实践，因为如果开发人员知道他们正在安全地获取一个元素，就不应该强迫他们处理一个可为空的值，同时，如果他们有任何不确定，他们应该使用`getOrNull `并正确地处理默认值。