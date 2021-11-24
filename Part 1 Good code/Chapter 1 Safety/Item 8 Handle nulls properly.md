## Item 8: Handle nulls properly

`null` means a lack of value. For property, it might mean that the value was not set or that it was removed. When a function returns `null` it might have a different meaning depending on the function:

- `String.toIntOrNull()` returns `null` when `String` cannot be correctly parsed to `Int`
- `Iterable<T>.firstOrNull(() -> Boolean)` returns `null` when there are no elements matching predicate from the argument. 

**In these and all other cases the meaning of `null` should be as clear as possible.** This is because nullable values must be handled, and it is the API user (programmer using API element) who needs to decide how it is to be handled. 

``` kotlin
val printer: Printer? = getPrinter()
printer.print() // Compilation Error

printer?.print() // Safe call
if (printer != null) printer.print() // Smart casting
printer!!.print() // Not-null assertion
```

In general, there are 3 ways of how nullable types can be handled. We can:

- Handling nullability safely using safe call `?.`, smart casting, Elvis operator, etc.
- Throw an error
- Refactor this function or property so that it won’t be nullable

Let’s describe them one by one.

### Handling nulls safely

As mentioned before, the two safest and most popular ways to handle nulls is by using safe call and smart casting:

``` kotlin
printer?.print() // Safe call
if (printer != null) printer.print() // Smart casting
```

In both those cases, the function `print` will be called only when `printer` is not `null`. This is the safest option from the application user point of view. It is also really comfortable for programmers. No wonder why this is generally the most popular way how we handle nullable values. 

Support for handling nullable variables in Kotlin is much wider than in other languages. One popular practice is to use the Elvis operator which provides a default value for a nullable type. It allows any expression including `return` and `throw` on its right side:

``` kotlin
val printerName1 = printer?.name ?: "Unnamed"
val printerName2 = printer?.name ?: return
val printerName3 = printer?.name ?: 
    throw Error("Printer must be named")
```

Many objects have additional support. For instance, as it is common to ask for an empty collection instead of `null`, there is `orEmpty` extension function on `Collection<T>?` returning not-nullable `List<T>`. There is also a similar function for `String?`.

Smart casting is also supported by Kotlin’s contracts feature that lets us smart cast in a function:

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

All those options should be known to Kotlin developers, and they all provide useful ways to handle nullability properly. 

### Defensive and offensive programming

Handling all possibilities in a correct way - like here not using `printer` when it is `null` - is an implementation of *defensive programming*. *Defensive programming* is a blanket term for various practices increasing code stability once the code is in production, often by defending against the currently impossible. It is the best way when there is a correct way to handle all possible situations. It wouldn’t be correct if we would expect that `printer` is not `null` and should be used. In such a case it would be impossible to handle such a situation safely, and we should instead use a technique called *offensive programming*. The idea behind *offensive programming* is that in case of an unexpected situation we complain about it loudly to inform the developer who led to such situation, and to force him or her to correct it. A direct implementation of this idea is `require`, `check`and `assert` presented in Item 5: Specify your expectations on arguments and state. It is important to understand that even though those two modes seem like being in conflict, they are not at all. They are more like yin and yang. Those are different modes both needed in our programs for the sake of safety, and we need to understand them both and use them appropriately. 



### Throw an error

One problem with safe handling is that if `printer` could sometimes be `null`, we will not be informed about it but instead `print` won’t be called. This way we might have hidden important information. If we are expecting `printer` to never be `null`, we will be surprised when the `print` method isn’t called. This can lead to errors that are really hard to find. When we have a strong expectation about something that isn’t met, it is better to throw an error to inform the programmer about the unexpected situation. It can be done using `throw`, as well as by using the not-null assertion `!!`, `requireNotNull`, `checkNotNull`, or other error throwing functions:



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

### The problems with the not-null assertion `!!`

The simplest way to handle nullable is by using not-null assertion `!!`. It is conceptually similar to what happens in Java - we think something is not `null` and if we are wrong, an NPE is thrown. **Not-null assertion `!!` is a lazy option. It throws a generic exception that explains nothing. It is also short and simple, which also makes it easy to abuse or misuse.** Not-null assertion `!!` is often used in situations where type is nullable but `null` is not expected. The problem is that even if it currently is not expected, it almost always can be in the future, and this operator only quietly hides the nullability.

A very simple example is a function looking for the largest among 4 arguments[9](chap65.xhtml#fn-maxOf). Let’s say that we decided to implement it by packing all those arguments into a list and then using `max` to find the biggest one. The problem is that it returns nullable because it returns `null`when the collection is empty. Though developer knowing that this list cannot be empty will likely use not-null assertion `!!`:

``` kotlin
fun largestOf(a: Int, b: Int, c: Int, d: Int): Int =
   listOf(a, b, c, d).max()!!
```

As it was shown to me by Márton Braun who is a reviewer of this book, even in such a simple function not-null assertion `!!` can lead to NPE. Someone might need to refactor this function to accept any number of arguments and miss the fact that collection cannot be empty:



``` kotlin
fun largestOf(vararg nums: Int): Int =
   nums.max()!!

largestOf() // NPE
```

Information about nullability was silenced and it can be easily missed when it might be important. Similarly with variables. Let’s say that you have a variable that needs to be set later but it will surely be set before the first use. Setting it to `null` and using a not-null assertion `!!` is not a good option:

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

It is not only annoying that we need to unpack those properties every time, but we also block the possibility for those properties to actually have a meaningful `null` in the future. Later in this item, we will see that the correct way to handle such situations is to use `lateinit` or `Delegates.notNull`.

**Nobody can predict how code will evolve in the future, and if you use not-null assertion `!!` or explicit error throw, you should assume that it will throw an error one day.** Exceptions are thrown to indicate something unexpected and incorrect (Item 7: Prefer null or a sealed result class result when the lack of result is possible). However, **explicit errors say much more than generic NPE and they should be nearly always preferred**.

Rare cases where not-null assertion `!!` does make sense are mainly a result of common interoperability with libraries in which nullability is not expressed correctly. When you interact with an API properly designed for Kotlin, this shouldn’t be a norm.

In general **we should avoid using the not-null assertion `!!`**. This suggestion is rather widely approved by our community. Many teams have the policy to block it. Some set the Detekt static analyzer to throw an error whenever it is used. I think such an approach is too extreme, but I do agree that it often is a code smell. **Seems like the way this operator looks like is no coincidence. `!!` seems to be screaming “Be careful” or “There is something wrong here”.**

### Avoiding meaningless nullability

Nullability is a cost as it needs to be handled properly and we **prefer avoiding nullability when it is not needed.** `null` might pass an important message, but we should avoid situations where it would seem meaningless to other developers. They would then be tempted to use the unsafe not-null assertion `!!` or forced to repeat generic safe handling that only clutters the code. We should avoid nullability that does not make sense for a client. The most important ways for that are:

- Classes can provide variants of functions where the result is expected and in which lack of value is considered and nullable result or a sealed result class is returned. Simple example is `get` and `getOrNull` on `List<T>`. Those are explained in detail in Item 7: Prefer `null` or a sealed result class result when the lack of result is possible.
- Use `lateinit` property or `notNull` delegate when a value is surely set before use but later than during class creation.
- **Do not return `null` instead of an empty collection.** When we deal with a collection, like `List<Int>?` or `Set<String>?`, `null` has a different meaning than an empty collection. It means that no collection is present. To indicate a lack of elements, use an empty collection. 
- Nullable enum and `None` enum value are two different messages. `null` is a special message that needs to be handled separately, but it is not present in the enum definition and so it can be added to any use-side you want.

Let’s talk about `lateinit` property and `notNull` delegate as they deserve a deeper explanation. 

### `lateinit` property and `notNull` delegate

It is not uncommon in projects to have properties that cannot be initialized during class creation, but that will surely be initialized before the first use. A typical example is when the property is set-up in a function called before all others, like in `@BeforeEach` in JUnit 5:

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

Casting those properties from nullable to not null whenever we need to use them is highly undesirable. It is also meaningless as we expect that those values are set before tests. The correct solution to this problem is to use `lateinit` modifier that lets us initialize properties later:

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

The cost of `lateinit` is that if we are wrong and we try to get value before it is initialized, then an exception will be thrown. Sounds scary but it is actually desired - we should use `lateinit` only when we are sure that it will be initialized before the first use. If we are wrong, we want to be informed about it. The main differences between `lateinit` and a nullable are:

- We do not need to “unpack” property every time to not-nullable
- We can easily make it nullable in the future if we need to use `null` to indicate something meaningful
- Once property is initialized, it cannot get back to an uninitialized state

**`lateinit` is a good practice when we are sure that a property will be initialized before the first use**. We deal with such a situation mainly when classes have their lifecycle and we set properties in one of the first invoked methods to use it on the later ones. For instance when we set objects in `onCreate` in an Android `Activity`, `viewDidAppear` in an iOS `UIViewController`, or `componentDidMount` in a React `React.Component`.

One case in which `lateinit` cannot be used is when we need to initialize a property with a type that, on JVM, associates to a primitive, like `Int`, `Long`, `Double` or `Boolean`. For such cases we have to use `Delegates.notNull` which is slightly slower, but supports those types:

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

These kinds of cases are often replaced with property delegates, like in the above example where we read the argument in `onCreate`, we could instead use a delegate that initializes those properties lazily:

``` kotlin
class DoctorActivity: Activity() {
   private var doctorId: Int by arg(DOCTOR_ID_ARG)
   private var fromNotification: Boolean by 
       arg(FROM_NOTIFICATION_ARG)
}
```

The property delegation pattern is described in detail in Item 21: Use property delegation to extract common property patterns. One reason why they are so popular is that they help us safely avoid nullability.