## 第8条：正确地处理`null`值

`null` 表示缺少值。对于属性，这可能意味着该值没有设置或已被删除。当函数返回 `null` 时，其含义可能因函数而异：

- `String.toIntOrNull()` 当 `String` 无法正确解析为 `Int` 时返回 `null`
- `Iterable<T>.firstOrNull(() -> Boolean)` 当没有元素符合参数传递的条件时返回 `null`

**在这些和所有其他情况下，`null` 的含义应尽可能清晰明确。** 这是因为可空值必须得到处理，而需要决定如何处理的是使用 API 元素的 API 用户（即程序员）。

``` kotlin
val printer: Printer? = getPrinter()
printer.print() // Compilation Error

printer?.print() // Safe call
if (printer != null) printer.print() // Smart casting
printer!!.print() // Not-null assertion
```

一般来说，处理可空类型有三种方法。我们可以：

- 使用安全调用 `?.`、智能转换、Elvis 运算符等安全地处理空值。
- 抛出错误。
- 重构函数或属性，使其不可为空。

我们逐一介绍它们。

### 安全地处理`null`值

如前所述，处理空值最安全、最常见的方式是使用安全调用和智能转换：

``` kotlin
printer?.print() // Safe call
if (printer != null) printer.print() // Smart casting
```

在这两种情况下，只有在 `printer` 不为 `null` 时才会调用 `print` 函数。从应用程序用户的角度来看，这是最安全的选择。对于程序员来说，这也非常方便。难怪这通常是我们处理可空值的最常用方式。

Kotlin 对处理可空变量的支持要比其他语言广泛得多。一种常见的做法是使用 Elvis 运算符，为可空类型提供默认值。它允许在其右侧使用任何表达式，包括 `return` 和 `throw`：

``` kotlin
val printerName1 = printer?.name ?: "Unnamed"
val printerName2 = printer?.name ?: return
val printerName3 = printer?.name ?: 
    throw Error("Printer must be named")
```

许多对象还提供了其他支持。例如，由于通常要求返回空集合而不是 `null`，因此在 `Collection<T>?` 上有一个 `orEmpty` 扩展函数，返回非空的 `List<T>`。对于 `String?`，也有类似的函数。

智能转换也受到 Kotlin 合同（contracts）功能的支持，它允许我们在函数中进行智能转换：

``` kotlin
println("What is your name?")
val name = readLine()
if (!name.isNullOrBlank()) {
   println("Hello ${name.toUpperCase()}")
}

val news: List<News>? = getNews()
if (!news.isNullOrEmpty()) {
   news.forEach { notifyUser(it) }
}
```

所有这些选项都应该为 Kotlin 开发人员所熟知，并且它们都提供了适当处理可空性的有用方法。

### 防御性编程和攻击性编程

以正确的方式处理所有可能情况，比如在 `printer` 为 `null` 时不使用它，是实现*防御性编程*的一种方式。*防御性编程*是一个广义的术语，指的是通过防范当前不可能的情况，从而提高代码在生产环境中的稳定性的各种实践方法。当有一种正确的方式来处理所有可能的情况时，这是最好的方式。如果我们期望 `printer` 不为 `null` 并且应该使用它，那么这种情况下，就需要进行防御性编程。

另一种处理空值的方式是使用*攻击性编程*的原则。攻击性编程是一种开发方法，旨在尽早发现和解决问题，以提高代码的可靠性和健壮性。在使用攻击性编程时，我们假设最坏的情况会发生，并采取相应的措施来处理它。


### 抛出异常

使用安全处理的一个问题是，如果`printer`有时可能为`null`，我们将不会收到任何通知，而是不会调用`print`方法。这样，我们可能会隐藏重要的信息。如果我们期望`printer`永远不会为`null`，当`print`方法没有被调用时，我们会感到惊讶。这可能导致非常难以找到的错误。当我们对某个未满足的强烈期望时，最好抛出错误以通知程序员发生了意外情况。可以使用`throw`关键字，以及使用非空断言`!!`、`requireNotNull`、`checkNotNull`或其他抛出错误的函数，如下所示：



``` kotlin
fun process(user: User) {
    requireNotNull(user.name)
    val context = checkNotNull(context)
    val networkService = 
        getNetworkService(context) ?: 
        throw NoInternetConnection()
    networkService.getData { data, userData ->
        show(data!!, userData!!)
    }
}
```

### 非空断言 `!!` 的问题

处理可为空值的最简单方法是使用非空断言 `!!`。它在概念上类似于Java中的行为-我们认为某个值不为`null`，如果我们错了，就会抛出空指针异常（NPE）。**非空断言 `!!` 是一种懒惰的选择。它只会抛出一个一般性的异常，没有提供任何详细信息。它也非常简短和简单，这也使得滥用或误用它变得容易。**非空断言 `!!` 经常用于类型可为空但不期望为`null`的情况下。问题是，即使当前不期望为`null`，但在将来几乎总是可能发生，而这个操作符只会悄悄地隐藏了可为空性。

一个非常简单的例子是一个函数查找4个参数中的最大值 9。假设我们决定通过将所有这些参数打包到列表中，然后使用 `max` 方法来找到最大值。问题是，它返回可为空值，因为当集合为空时返回`null`。尽管开发人员知道该列表不可能为空，但很可能会使用非空断言`!!`：

``` kotlin
fun largestOf(a: Int, b: Int, c: Int, d: Int): Int =
   listOf(a, b, c, d).max()!!
```

正如这本书的审阅者Márton Braun向我展示的，即使在这样一个简单的函数中，非空断言`!!`也可能导致空指针异常。有人可能需要重构此函数以接受任意数量的参数，并错过集合不能空的事实：



``` kotlin
fun largestOf(vararg nums: Int): Int =
   nums.max()!!

largestOf() // NPE
```

可空性的信息被消除，它很容易被忽视，而这可能是重要的。类似的情况也适用于变量。假设你有一个变量，在第一次使用之前需要设置它，但它肯定会在第一次使用之前设置。将其设置为null并使用非空断言!!不是一个好的选择：

``` kotlin
class UserControllerTest {

    private var dao: UserDao? = null
    private var controller: UserController? = null

    @BeforeEach
    fun init() {
        dao = mockk()
        controller = UserController(dao!!)
    }

    @Test
    fun test() {
        controller!!.doSomething()
    }

}
```

不仅需要每次对这些属性进行解包，而且还阻止了这些属性在将来实际上可能具有有意义的`null`值的可能性。稍后在本节中，我们将看到处理这种情况的正确方法是使用`lateinit`或`Delegates.notNull()`。

没有人能够预测代码在未来会如何发展，如果使用非空断言`!!`或显式错误抛出，就应该假设它将来某一天会抛出错误。异常被抛出来表示出现了意外和不正确的情况（第7条：当不能返回预期结果时，优先使用`null` o或`Failure` 作为返回值）。然而，显式错误信息比一般性的空指针异常更能传达信息，几乎总是应该优先选择使用显式错误。

非空断言`!!`确实有意义的罕见情况主要是由于与不正确地表达了可空性的库的常见互操作性。当与为Kotlin正确设计的API进行交互时，这不应该成为一种常态。

总体而言，我们应该避免使用非空断言`!!`。这个建议在我们的社区中得到了广泛认可。许多团队都有阻止使用它的策略。有些团队将Detekt静态分析器设置为在使用时抛出错误。我认为这种方法过于极端，但我同意它通常是一种代码异味。似乎这个操作符的样子并非巧合。`!!`似乎在呼喊着“小心”或“这里有问题”。

### 避免无意义的可空性

可空性是一种成本，因为它需要被正确处理，我们**更倾向于在不需要时避免可空性**。`null`可能传递了重要的信息，但我们应该避免出现对其他开发人员来说看起来毫无意义的情况。否则，他们可能会诱惑使用不安全的非空断言`!!`，或被迫重复通用的安全处理，从而只会使代码变得混乱。我们应该避免对客户端来说没有意义的可空性。实现这一点的最重要方法有：

- 类可以提供函数的变体，其中期望返回结果，并将缺少值视为可接受的情况，返回可为空的结果或一个密封结果类。`List<T>`上的`get`和`getOrNull`就是一个简单的例子。这些在“条款7：当可能缺少结果时，优先使用null或密封的结果类”中有详细解释。
- 在确保在使用之前但在类创建期间之后将值设置的情况下，使用`lateinit`属性或`notNull`委托。
- **不要返回`null`而不是空集合**。当我们处理集合时，例如`List<Int>?`或`Set<String>?`，`null`和空集合具有不同的含义。它表示没有集合存在。要表示缺少元素，请使用空集合。
- 可为空的枚举和`None`枚举值是两个不同的消息。`null`是一个需要单独处理的特殊消息，但它在枚举定义中不存在，因此可以添加到任何您想要的使用方。

让我们谈谈`lateinit`属性和`notNull`委托，因为它们值得更深入的解释。

### `lateinit`属性和`notNull`委托

在项目中，有些属性在类创建期间无法初始化，但在第一次使用之前肯定会被初始化。一个典型的例子是当属性在其他所有函数之前设置，比如在JUnit 5的`@BeforeEach`中：

``` kotlin
class UserControllerTest {

    private var dao: UserDao? = null
    private var controller: UserController? = null

    @BeforeEach
    fun init() {
        dao = mockk()
        controller = UserController(dao!!)
    }

    @Test
    fun test() {
        controller!!.doSomething()
    }

}
```

每次需要使用这些属性时，将其从可空类型转换为非空类型是非常不理想的。而且这样做是没有意义的，因为我们期望这些值在测试之前被设置。解决这个问题的正确方法是使用`lateinit`修饰符，它允许我们稍后初始化属性：

``` kotlin
class UserControllerTest {
   private lateinit var dao: UserDao
   private lateinit var controller: UserController

   @BeforeEach
   fun init() {
       dao = mockk()
       controller = UserController(dao)
   }

   @Test
   fun test() {
       controller.doSomething()
   }
}
```

`lateinit`的代价是，如果我们错误地在未初始化之前尝试获取值，那么将抛出异常。听起来有些可怕，但这实际上是期望的行为 - 我们只应该在确保在第一次使用之前它会被初始化的情况下使用`lateinit`。如果我们错了，我们希望得到相应的提示。`lateinit`和可空类型之间的主要区别有：

- 我们不需要每次都将属性“解包”为非空类型
- 如果将来需要使用`null`表示某个有意义的内容，我们可以轻松地将其设置为可空类型
- 一旦属性被初始化，就不能返回到未初始化的状态

**当我们确信一个属性在第一次使用之前会被初始化时，使用`lateinit`是一个良好的做法**。我们通常在类有其生命周期的情况下处理这种情况，我们在最早调用的方法之一中设置属性，以便在后续的方法中使用。例如，在Android的`Activity`的`onCreate`中设置对象，在iOS的`UIViewController`的`viewDidAppear`中设置对象，或者在React的`React.Component`的`componentDidMount`中设置对象。

`lateinit`无法使用的一种情况是当我们需要将属性初始化为JVM上与原语关联的类型，例如`Int`、`Long`、`Double`或`Boolean`。对于这种情况，我们必须使用`Delegates.notNull`，它的速度略慢，但支持这些类型：

``` kotlin
class DoctorActivity: Activity() {
   private var doctorId: Int by Delegates.notNull()
   private var fromNotification: Boolean by 
       Delegates.notNull()

   override fun onCreate(savedInstanceState: Bundle?) {
       super.onCreate(savedInstanceState)
       doctorId = intent.extras.getInt(DOCTOR_ID_ARG)
       fromNotification = intent.extras
          .getBoolean(FROM_NOTIFICATION_ARG)
   }
}
```

这种情况经常使用属性委托来替代，就像上面的例子中，我们在`onCreate`中读取参数时，可以使用一个委托来延迟初始化这些属性：

``` kotlin
class DoctorActivity: Activity() {
   private var doctorId: Int by arg(DOCTOR_ID_ARG)
   private var fromNotification: Boolean by 
       arg(FROM_NOTIFICATION_ARG)
}
```

属性委托模式在“Item 21: Use property delegation to extract common property patterns”中有详细描述。它们之所以如此受欢迎的一个原因是，它们帮助我们安全地避免了可空性。