## 第17条：考虑命名参数

当你阅读代码时，参数的含义并不总是清晰的。看看下面的例子：

``` kotlin
val text = (1..10).joinToString("|")
```

`"|"`是什么？如果你对`joinToString`很熟悉，你就知道它是`separator`。虽然它也可能是`prefix`。这一点完全不清楚。我们可以通过明确那些值并不清楚其含义的参数来使其更易读。最好的方法就是使用命名参数：

``` kotlin
val text = (1..10).joinToString(separator = "|")
```

我们也可以通过命名变量达到类似的效果：

``` kotlin
val separator = "|"
val text = (1..10).joinToString(separator)
```

尽管命名参数更可靠。变量名指定了开发者的意图，但并不一定是正确的。如果开发者犯了错误，把变量放在了错误的位置呢？如果参数的顺序改变了呢？命名参数可以保护我们免受这种情况的影响，而命名值则不能。这就是为什么当我们有命名值时，仍然有理由使用命名参数：

``` kotlin
val separator = "|"
val text = (1..10).joinToString(separator = separator)
```

### 我们应该何时使用命名参数？

显然，命名参数更长，但它们有两个重要的优点：

- 名称指示预期的值。
- 它们更安全，因为它们独立于顺序。

参数名称不仅对使用此函数的开发者重要，对阅读它如何被使用的人也重要。看看这个调用：

``` kotlin
sleep(100)
```

它会睡多久？100毫秒？可能是100秒？我们可以使用命名参数来澄清它：

``` kotlin
sleep(timeMillis = 100)
```

在这种情况下，这并不是唯一的澄清选项。在像Kotlin这样的静态类型语言中，当我们传递参数时，保护我们的第一个机制是参数类型。我们可以在这里使用它来表达关于时间单位的信息：

``` kotlin
sleep(Millis(100))
```

或者，我们可以使用扩展属性来创建类似DSL的语法：

``` kotlin
sleep(100.ms)
```

类型是传递此类信息的好方法。如果你关心效率，可以使用*第46条：对具有功能类型参数的函数使用内联修饰符*中描述的内联类。它们帮助我们保证参数的安全性，但并不能解决所有问题。一些参数可能仍然不清楚。一些参数可能仍然被放在错误的位置。这就是为什么我仍然建议考虑使用命名参数，特别是对于参数：

- 带有默认参数的，
- 与其他参数类型相同的，
- 功能类型的，如果它们不是最后一个参数。

### 带有默认参数的参数

当一个属性有一个默认参数时，我们几乎总是应该通过名称使用它。这种可选参数比那些必需的参数更常更改。我们不想错过这样的变化。函数名称通常指示其非可选参数是什么，但不是其可选参数是什么。这就是为什么命名可选参数通常更安全、更清洁的原因。

### 许多类型相同的参数

正如我们所说，当参数有不同的类型时，我们通常可以避免将参数放在错误的位置。但是，当一些参数有相同的类型时，就没有这样的自由了。

``` kotlin
fun sendEmail(to: String, message: String) { /*...*/ }
```

对于这样的函数，最好使用名称来澄清参数：

``` kotlin
sendEmail(
   to = "contact@kt.academy",
   message = "Hello, ..."
)
```

### 功能类型的参数

最后，我们应该特别对待具有函数类型的参数。在Kotlin中，这样的参数有一个特殊的位置：最后一个位置。有时，函数名描述了一个函数类型的参数。例如，当我们看到`repeat`时，我们期望在其后的lambda是应该被重复的代码块。当你看到`thread`时，直观地认为在其后的块是这个新线程的主体。这样的名称只描述了在最后位置使用的函数。

``` kotlin
thread {
   // ...
}
```

所有其他具有函数类型的参数都应该被命名，因为很容易误解它们。例如，看看这个简单的视图DSL：

``` kotlin
val view = linearLayout {
   text("Click below")
   button({ /* 1 */ }, { /* 2 */ })
}
```

哪个函数是这个构建器的一部分，哪个是点击监听器？我们应该通过命名监听器并将构建器移出参数来澄清这一点：

``` kotlin
val view = linearLayout {
   text("Click below")
   button(onClick = { /* 1 */ }) {
      /* 2 */
   }
}
```

函数类型的多个可选参数可能会特别令人困惑：

``` kotlin
fun call(before: ()->Unit = {}, after: ()->Unit = {}){
   before()
   print("Middle")
   after()
}

call({ print("CALL") }) // CALLMiddle
call { print("CALL") }  // MiddleCALL
```

为了防止这种情况，当没有一个具有特殊含义的函数类型的参数时，给它们全部命名：

``` kotlin
call(before = { print("CALL") }) // CALLMiddle
call(after = { print("CALL") })  // MiddleCALL
```

对于反应式库来说，这一点尤其重要。例如，在RxJava中，当我们订阅一个`Observable`时，我们可以设置应该被调用的函数：

- 在每个接收到的项目上
- 在出现错误的情况下
- 在observable完成后

在Java中，我经常看到人们使用lambda表达式来设置它们，并在注释中指定每个lambda表达式是哪个方法。

``` kotlin
// Java
observable.getUsers()
       .subscribe((List<User> users) -> { // onNext
           // ...
       }, (Throwable throwable) -> { // onError
           // ...
       }, () -> { // onCompleted
           // ...
       });
```

在 Kotlin 中，我们可以进一步使用命名参数：

``` kotlin
observable.getUsers()
   .subscribeBy(
       onNext = { users: List<User> ->
           // ...
       },
       onError = { throwable: Throwable ->
           // ...
       },
       onCompleted = {
           // ...
       })
```

注意，我将函数名从 `subscribe` 更改为 `subscribeBy`。这是因为 `RxJava` 是用 Java 编写的，**我们在调用 Java 函数时不能使用命名参数**。这是因为 Java 不保留有关函数名称的信息。为了能够使用命名参数，我们通常需要为这些函数制作我们的 Kotlin 包装器（作为这些函数的替代方案的扩展函数）。

### 总结

命名参数不仅在我们需要跳过一些默认值时有用。它们对于阅读我们代码的开发人员来说是重要的信息，它们可以提高我们代码的安全性。当我们有更多相同类型的参数（或具有功能类型的参数）和可选参数时，我们应该考虑它们。当我们有多个具有功能类型的参数时，它们几乎总是应该被命名。例外情况是最后一个函数参数，当它具有特殊含义，如在 DSL 中。