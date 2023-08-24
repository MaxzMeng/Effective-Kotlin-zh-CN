## 第16条：属性应代表状态，而不是行为

Kotlin的属性看起来类似于Java的字段，但它们实际上代表了一个不同的概念。

``` kotlin
// Kotlin属性
var name: String? = null

// Java字段
String name = null;
```

尽管它们可以以相同的方式使用，来存储数据，我们需要记住，属性具有更多的功能。首先，它们总是可以有自定义的设置器和获取器：

``` kotlin
var name: String? = null
   get() = field?.toUpperCase()
   set(value) {
       if(!value.isNullOrBlank()) {
           field = value
       }
   }
```

你可以看到我们在这里使用了`field`标识符。这是对让我们在此属性中保存数据的后备字段的引用。这样的后备字段是默认生成的，因为设置器和获取器的默认实现使用它们。我们也可以实现不使用它们的自定义访问器，在这种情况下，属性将根本没有`field`。例如，Kotlin属性可以只使用获取器定义为只读属性`val`：

``` kotlin
val fullName: String
   get() = "$name $surname"
```

对于可读写的属性`var`，我们可以通过定义获取器和设置器来创建一个属性。这样的属性被称为*派生属性*，并且并不少见。它们是Kotlin中所有属性默认被封装的主要原因。只需想象一下，你需要在你的对象中保存一个日期，你使用了Java stdlib的`Date`。然后在某个时候，由于某种原因，对象无法再存储这种类型的属性。可能是因为序列化问题，或者可能是因为你将此对象提升到了一个公共模块。问题在于，这个属性已经在你的项目中被引用。有了Kotlin，这就不再是问题，因为你可以将你的数据移动到一个单独的属性`millis`，并修改`date`属性，使其不再保存数据，而是包装/解包那个其他属性。

``` kotlin
var date: Date
   get() = Date(millis)
   set(value) {
       millis = value.time
   }
```

属性不需要字段。从概念上讲，它们代表访问器（`val`的 getter，`var`的 getter 和 setter）。这就是我们可以在接口中定义它们的原因：

``` kotlin
interface Person {
   val name: String
}
```

这意味着这个接口承诺有一个 getter。我们也可以覆盖属性：

``` kotlin
open class Supercomputer {
   open val theAnswer: Long = 42
}

class AppleComputer : Supercomputer() {
   override val theAnswer: Long = 1_800_275_2273
}
```

出于同样的原因，我们可以委托属性：

``` kotlin
val db: Database by lazy { connectToDb() }
```

属性委托在第21条：使用属性委托提取常见的属性模式中详细描述。因为属性本质上是函数，我们也可以创建扩展属性：

``` kotlin
val Context.preferences: SharedPreferences
   get() = PreferenceManager
       .getDefaultSharedPreferences(this)

val Context.inflater: LayoutInflater
   get() = getSystemService(
       Context.LAYOUT_INFLATER_SERVICE) as LayoutInflater

val Context.notificationManager: NotificationManager
   get() = getSystemService(Context.NOTIFICATION_SERVICE) 
       as NotificationManager
```

如你所见，**属性代表访问器，而不是字段**。这样，它们可以代替一些函数，但我们应该小心我们用它们来做什么。属性不应该用来表示像下面的例子中的算法行为：

``` kotlin
// DON’T DO THIS!
val Tree<Int>.sum: Int
   get() = when (this) {
       is Leaf -> value
       is Node -> left.sum + right.sum
   }
```

"这里的 `sum` 属性遍历所有元素，因此它代表了算法行为。因此，这个属性具有误导性：对于大型集合，寻找答案可能在计算上很重，这完全出乎预料。这不应该是一个属性，而应该是一个函数：

``` kotlin
fun Tree<Int>.sum(): Int = when (this) {
   is Leaf -> value
   is Node -> left.sum() + right.sum()
}
```

一般规则是，**我们只应该用它们来表示或设置状态，不应该涉及其他逻辑**。一个有用的启发式规则来决定是否应该是一个属性是：如果我将这个属性定义为一个函数，我会在它前面加上get/set吗？如果不是，那么它可能不应该是一个属性。更具体地说，以下是我们不应该使用属性，而应该使用函数的最典型情况：

- **操作在计算上昂贵或计算复杂度高于O(1)** - 用户不期望使用属性可能会很昂贵。如果是这样，使用函数更好，因为它传达了可能会这样，用户可能会节俭地使用它，或者开发者可能会考虑缓存它。

- **它涉及业务逻辑（应用程序如何行动）** - 当我们阅读代码时，我们不期望一个属性可能会做任何超过简单动作的事情，比如记录，通知监听器，或者更新一个绑定的元素。

- **它不是确定性的** - 连续两次调用成员会产生不同的结果。

- **它是一个转换，比如** `Int.toDouble()` - 按照惯例，转换是一个方法或一个扩展函数。使用属性会像引用某个内部部分，而不是包装整个对象。

- **获取器不应改变属性状态** - 我们期望我们可以自由地使用获取器，而不用担心属性状态的修改。

  

例如，计算元素的总和需要遍历所有元素（这是行为，而不是状态），并且具有线性复杂性。因此，它不应该是一个属性，并且在标准库中被定义为一个函数：

``` kotlin
val s = (1..100).sum()
```

另一方面，为了获取和设置状态，我们在Kotlin中使用属性，除非有充分的理由，否则我们不应该涉及到函数。我们使用属性来表示和设置状态，如果你以后需要修改它们，使用自定义的获取器和设置器：

``` kotlin
// DON’T DO THIS!
class UserIncorrect {
   private var name: String = ""
  
   fun getName() = name
  
   fun setName(name: String) {
       this.name = name
   }
}

class UserCorrect {
   var name: String = ""
}
```

一个简单的经验法则是，**属性描述和设置状态，而函数描述行为**。

