## 第8条: 正确的处理空值

`null`意味着缺乏价值的，对属性来说，它可能意味着没有被赋值或者被移除了 ，当函数返回`null`时，它可能具有不同的含义，具体取决于函数：

- `String.toIntOrNull()` - 当字串无法正确的被解析为`Int`时返回`null`
- `Iterable<T>.firstOrNull(() -> Boolean)` - 当参数中没有与述句(*意指first*)匹配的元素时返回`null`

*不只上述情况，还有其他所有的情况下，`null`的含义应该要尽可能地清楚*，这是因为`null`应该要被处理，而API使用者(使用API元素的开发者)需要决定如何处理它

``` kotlin
val printer: Printer? = getPrinter()
printer.print() // Compilation Error

printer?.print() // Safe call
if (printer != null) printer.print() // Smart casting
printer!!.print() // Not-null assertion
```

一般来说，有三种方法可以处理可空型别，我们可以：

- 利用*safe-call(`?.`)，smart-casting，Elvis operator(?:)*等，来安全的处理可空性
- 抛出一个错误
- 重构这个函数或属性，使它不为`null`

让我们一一描述它们

### 安全的处理空值

如之前所提，处理空值的两种最安全与最受欢迎的方法是使用`安全调用(safe-call)`和`智能转换(smart-casting)`

``` kotlin
printer?.print() // Safe call
if (printer != null) printer.print() // Smart casting
```

上方例子，`print`这个函数只有在`printer`不为空时才会被调用，从应用程式使用者的观点来说，这是最安全的选项，对开发人员来说也是蛮舒适的方法，这也难怪为何他们通常是处理可空值最流行的方法

Kotlin对处理可空变数的支援比其他语言更广泛，一种流行的做法是使用`Elvis`运算子，它为可空类型提供了预设值，它允许任何表达式存在于运算子右侧，包含`return`与`throw`:

``` kotlin
val printerName1 = printer?.name ?: "Unnamed"
val printerName2 = printer?.name ?: return
val printerName3 = printer?.name ?: 
    throw Error("Printer must be named")
```

许多物件都有额外的支援，例如：我们通常会要求一个集合是`空集合(empty)`而非`null`，而Collection\<T>?.orEmpty()这个扩展函数可以返回一个不为空的List，

在Kotlin的`Contracts(契约)`功能也支援智能转换(`smart-casting`)，让我们可以在函数中使用智能转换

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

Kotlin的开发人员应该都要知道有这些选项，它们都提供了有用的方法来正确处理可空性

### 防御式与进攻式编程

以正确的方式处理各种可能性- 如同我们稍早提到的：在printer为空时不使用它，这是一种防御性编程的实作，防御性编程是各种实践的总称，一旦程式码投入生产(意指在专案中使用它)，就会提高程式码的稳定性，通常是透过防御当前不可能的事情。当有正确的方法来处理所有可能的情况时，这是最好的方法，如果我们期望`printer`是不为空且应该被使用(因为`printer`本身是可空的变数)，那将会是不正确的，这种状况下，不可能安全的处理这种情况，而是使用一种称之为*攻击性编程*的技术，攻击性编程背后的想法是：如果出现非预期的情况，我们会大声抱怨它以通知导致这种状况的开发人员，进而强迫他(她)修正这个错误，而实作这个想法就是透过`require`、`check`与`assert`来达成（第5项:Specify your expectations on arguments and state）。更重要的是要理解，即使这两种模式(`攻击与防御编程`)看似存在冲突，但它们根本不冲突，它们更像*阴阳(一体两面的事物)*，为了安全起见，我们的程式码中都需要这些不同的模式，我们需要了解并正确的使用它们



### 抛出一个错误

O安全处理的一个问题是：如果`printer`有时可能为空时，我们不会被知会，而是不会调用`print`方法，这样子我们可能隐藏了重要讯息，如果我们预期`printer`永远不会为空，那么当`print`没有被调用时，我们会很惊讶，这可能会导致难以发现的错误，当我们对未能达成的事情抱有强烈期望时，最好是抛出错误以告知开发人员这个非预期的的状况，可以使用`throw`来完成，也可以用`非空断言(!!)`、`requireNotNull`、`checkNotNull`或其他错误抛出函数来完成:

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

### 非空断言(`!!`)的问题

处理可为空的最简单方法是使用`非空断言(!!)`，它在概念上类似于Java中发生的事情：我们认为某些东西不是空的，如果我们错了，就会抛出NPE(NullPointerException)。非空断言(!!)是一种懒惰的选项，它抛出了一个没有解释的通用异常，它也很简短，也很容易被滥用与误用。非空断言(!!)通常用于类型可为空但不希望为空的情境，但问题是，即使当前不是我们预期的，它几乎总是可以在未来出现，而这个运算子只是悄悄隐藏了可空性

一个非常简单的例子是一个函数在四个参数中找寻最大值，假设我们决定透过将这些参数通通打包到一个List中，然后使用`max`方法找到最大值，但问题是它(*max()这个方法，注：猜测应该是kotlin 1.3x版本前*)返回`nullable` ，因为它在集合为空时返回null，尽管开发人员知道此List不能为空，可能还是会使用非空断言(!!)：

``` kotlin
fun largestOf(a: Int, b: Int, c: Int, d: Int): Int =
   listOf(a, b, c, d).max()!!
```

正如本书的审稿人`Márton Braun`向我展示的那样，即使在这样一个简单的函数非空断言中，也可能导致NPE，有人可能需要重构此函数以接受任意数量的变数，而忽略了集合不能为空的事实：


``` kotlin
fun largestOf(vararg nums: Int): Int =
   nums.max()!!

largestOf() // NPE
```

I关于可空性的讯息被静默了，当它可能很重要时很容易被遗漏，跟变数类似，假设有一个变数需要稍后再设定，但肯定会在第一次使用前设定，将其设定为`null`并使用非空断言(`!!`)不是一个好选项：

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

我们每次都需要解包这些属性不仅很烦人，而且我们还阻止了这些属性在将来实际上具有意义的null的可能性，稍后，我们将看到处理这种情况的正确方法，是使用`lateinit`跟`Delegates.notNull`

*没有人可以预测程式码在未来会如何演变，如果你使用非空断言(!!)或显式错误抛出，你应该假设它有一天会抛出错误。*，抛出异常是为了表明非预期与错误(第7条:Prefer null or Failure result when the lack of result is possible)，然而，显式错误比通用NPE说明的要多得多，它们几乎总是首选。

有些使用非空断言的罕见情况是合理的，主要是与library中未正确表达可空性的共同互操作性(`common interoperability`)的结果，当你跟为Kotlin设定的API互动时，这不应该成为一种规范

通常我们应该避免使用非空断言(!!)，这个建议得到了我们社群广泛的认可，许多团队都有阻止它的政策，有些人将`Detekt静态分析器`设置在使用非空断言时抛出错误，我认为这样的做法太极端了，但我确实同意这通常是一种程式码异味(`code smell:可能导致深层次问题的症状`)，这个运算符的样子看起来也并非巧合，*!!* 符号似乎在尖叫着说“小心”或者“这里有问题”

### 避免无意义的可空性

可空性是一种成本，因为它需要被正确的处理，我们更倾向于在不需要时*避免使用可空性*，null可能会传递重要讯息，但我们应该避免让它在其他开发人员眼里是没有意义的情况，在这种情况下，开发者们会更倾向于使用不安全的非空断言(!!)或被迫重复使用只会让程式码更混乱的安全处理，我们应该避免那些对客户端没有意义的可空性，最重要的方法是：

- 类别可以提供预期结果的函数变体，其中缺少值(lack of value)我们考虑使用`可空的结果`或者`密封的Result类别`来返回，简单的例子是List的`get`和`getOrNull<T>`，这些在Item7有详细的解释，当可能缺少结果时，首选null或密封Result类别当作返回的结果
- 当你确定一个数值会在建立类别之后且在使用它之前设定，那么请使用`lateinit`或者`notNull委托`
- 当我们在处理集合时，返回`空集合`而不是null，像是List\<Int>? or Set\<String>?，null与空集合具有不同的含义，null意味着集合不存在，为了要表示缺少元素，请使用空集合
- `可空列举`跟`没有列举值`是两种不同的讯息，null是需要单独处理的特殊消息，但null并不存在于列举(`enum`)的定义中，因此可以将其添加到你想要的任何使用端

让我们来谈谈`lateinit`属性和`notNull`委托，因为它们值得更深入的解释

### `lateinit` 属性和 `notNull` 委托

在专案中，无法在类别建立期间初始化的属性并不少见，但这肯定会在第一次使用前被初始化，一个典型的例子是在一个函数内设置属性并且在调用其他所有函数之前呼叫这个设置属性的函数，例如在JUnit 5中的@BeforeEach中：

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

每当我们需要使用它们时，将这些属性从可空转换为不可空是非常不可取也毫无意义的，因为我们期望这些值是在测试之前设置的，正确的解决方案是使用`lateinit`修饰符，它可以让我们稍候再初始化属性：

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

`lateinit`的代价是如果我们错了，我们试图在初始化之前取值，那么它将会抛出异常，听起来很吓人，但实际上是需要的，只有当我们确定它会在第一次使用之前被初始化时，我们才应该使用`lateinit`，如果我们错了，我们希望被告知，`lateinit`与`可空性`最主要的差别是：

- 我们不需要每次都将属性`解包(unpack)`为不可空
- 我们如果在未来需要使用null来表示某些有意义的东西，我们可以轻易的将该属性设定为null
- 一旦属性被初始化之后，它就无法回到未初始化的状态

当我们确定属性会在第一次使用之前被初始化时，`lateinit`是一种很好的做法，我们主要处理的情况是当类别具有其生命周期，且我们在第一个调用的方法之中设置属性，并在之后的方法里使用这个属性，例如：当我们在Android Activity的onCreate中设置物件时，IOS UIViewController中的 viewDidAppear，或React React.Component中的componentDidMount

不能使用`lateinit`的一种情况是：当我们需要在JVM上初始化原生资料型别(像是：`Int`,`Long`,`Double`,`Boolean`)的属性，对于这种情况，我们必须使用`Delegates.notNull`，它稍微慢一些，但支援这些基本型态：

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

这些情况通常被属性委托替代，就像上述例子中我们在onCreate读取餐数一样，我们可以改用延迟初始化这些属性的委托：

``` kotlin
class DoctorActivity: Activity() {
   private var doctorId: Int by arg(DOCTOR_ID_ARG)
   private var fromNotification: Boolean by 
       arg(FROM_NOTIFICATION_ARG)
}
```

属性委托模式之后会在Item 21有详细的说明。 *它们之所以这么受欢迎是因为它们可以帮助我们安全的避免可空性*