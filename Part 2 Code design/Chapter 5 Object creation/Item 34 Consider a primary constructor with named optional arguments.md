## Item 34: Consider a primary constructor with named optional arguments

When we define an object and specify how it can be created, the most popular option is to use a primary constructor:

``` kotlin
class User(var name: String, var surname: String)
val user = User("Marcin", "Moskała")
```

Not only are primary constructors very convenient, but in most cases, it is actually very good practice to build objects using them. It is very common that we need to pass arguments that determine the object’s initial state, as illustrated by the following examples. Starting from the most obvious one: data model objects representing data[2](chap65.xhtml#fn-footnote_510_note). For such object, whole state is initialized using constructor and then hold as properties.

``` kotlin
data class Student(
   val name: String,
   val surname: String,
   val age: Int
)
```

Here’s another common example, in which we create a presenter[3](chap65.xhtml#fn-footnote_511_note) for displaying a sequence of indexed quotes. There we inject dependencies using a primary constructor:

``` kotlin
class QuotationPresenter(
       private val view: QuotationView,
       private val repo: QuotationRepository
) {
   private var nextQuoteId = -1

   fun onStart() {
       onNext()
   }

   fun onNext() {
       nextQuoteId = (nextQuoteId + 1) % repo.quotesNumber
       val quote = repo.getQuote(nextQuoteId)
       view.showQuote(quote)
   }
}
```

Note that `QuotationPresenter` has more properties than those declared on a primary constructor. In here `nextQuoteId` is a property always initialized with the value `-1`. This is perfectly fine, especially when the initial state is set up using default values or using primary constructor parameters.

To better understand why the primary constructor is such a good alternative in the majority of cases, we must first consider common Java patterns related to the use of constructors:

- the telescoping constructor pattern
- the builder pattern

We will see what problems they do solve, and what better alternatives Kotlin offers. 

### Telescoping constructor pattern

The telescoping constructor pattern is nothing more than a set of constructors for different possible sets of arguments:

``` kotlin
class Pizza {
   val size: String
   val cheese: Int
   val olives: Int
   val bacon: Int

   constructor(size: String, cheese: Int, olives: Int, 
bacon: Int) {
       this.size = size
       this.cheese = cheese
       this.olives = olives
       this.bacon = bacon
   }
   constructor(size: String, cheese: Int, olives: Int):
this(size, cheese, olives, 0)
   constructor(size: String, cheese: Int): 
this(size, cheese, 0)
   constructor(size: String): this(size, 0)
}
```

Well, this code doesn’t really make any sense in Kotlin, because instead we can use default arguments:

``` kotlin
class Pizza(
       val size: String,
       val cheese: Int = 0,
       val olives: Int = 0,
       val bacon: Int = 0
)
```

Default values are not only cleaner and shorter, but their usage is also more powerful than the telescoping constructor. We can specify just `size` and `olives`:

``` kotlin
val myFavorite = Pizza("L", olives = 3)
```

We can also add another named argument either before or after olives:

``` kotlin
val myFavorite = Pizza("L", olives = 3, cheese = 1)
```

As you can see, default arguments are more powerful than the telescoping constructor because:

- We can set any subset of parameters with default arguments we want.
- We can provide arguments in any order.
- We can explicitly name arguments to make it clear what each value means.

The last reason is quite important. Think of the following object creation:

``` kotlin
val villagePizza = Pizza("L", 1, 2, 3)
```

It is short, but is it clear? I bet that even the person who declared the pizza class won’t remember in which position bacon is, and in which position cheese can be found. Sure, in an IDE we can see an explanation, but what about those who just scan code or read it on Github? When arguments are unclear, we should explicitly say what their names are using *named arguments*:

``` kotlin
val villagePizza = Pizza(
   size = "L", 
   cheese = 1, 
   olives = 2, 
   bacon = 3
)
```

As you can see, **constructors with default arguments surpass the telescoping constructor pattern**. Though there are more popular construction patterns in Java, and one of them is the Builder Pattern. 

### Builder pattern

Named parameters and default arguments are not allowed in Java. This is why Java developers mainly use the builder pattern. It allows them to:

- name parameters,
- specify parameters in any order,
- have default values.

Here is an example of a builder defined in Kotlin:

``` kotlin
class Pizza private constructor(
       val size: String,
       val cheese: Int,
       val olives: Int,
       val bacon: Int
) {
   class Builder(private val size: String) {
       private var cheese: Int = 0
       private var olives: Int = 0
       private var bacon: Int = 0

       fun setCheese(value: Int): Builder = apply {
           cheese = value 
       }
       fun setOlives(value: Int): Builder = apply {
           olives = value
       }

       fun setBacon(value: Int): Builder = apply {
           bacon = value
       }

       fun build() = Pizza(size, cheese, olives, bacon)
   }
}
```

With the builder pattern, we can set those parameters as we want, using their names:

``` kotlin
val myFavorite = Pizza.Builder("L").setOlives(3).build()

val villagePizza = Pizza.Builder("L")
       .setCheese(1)
       .setOlives(2)
       .setBacon(3)
       .build()
```

As we’ve already mentioned, these two advantages are fully satisfied by Kotlin default arguments and named parameters:

``` kotlin
val villagePizza = Pizza(
   size = "L", 
   cheese = 1, 
   olives = 2, 
   bacon = 3
)
```

Comparing these two simple usages, you can see the advantages of named parameters over the builder:

- It’s shorter — a constructor or factory method with default arguments is much easier to implement than the builder pattern. It is a time-saver both for the developer who implements this code and for those who read it. It is a significant difference because the builder pattern implementation can be time-consuming. Any builder modification is hard as well, for instance, changing a name of a parameter requires not only name change of the function used to set it, but also name of parameter in this function, body of this function, internal field used to keep it, parameter name in the private constructor etc. 
- It’s cleaner — when you want to see how an object is constructed, all you need is in a single method instead of being spread around a whole builder class. How are objects held? Do they interact? These are questions that are not so easy to answer when we have a big builder. On the other hand, class creation is usually clear on a factory method.
- Offers simpler usage - the primary constructor is a built-in concept. The builder pattern is an artificial concept and it requires some knowledge about it. For instance, a developer can easily forget to call the `build` function (or in other cases `create`).
- No problems with concurrence —this is a rare problem, but function parameters are always immutable in Kotlin, while properties in most builders are mutable. Therefore it is harder to implement a thread-safe build function for a builder.

It doesn’t mean that we should always use a constructor instead of a builder. Let’s see cases where different advantages of this pattern shine.

Builders can require a set of values for a name (`setPositiveButton`, `setNegativeButton`, and `addRoute`), and allows us to aggregate (`addRoute`):

``` kotlin
val dialog = AlertDialog.Builder(context)
       .setMessage(R.string.fire_missiles)
       .setPositiveButton(R.string.fire, { d, id ->
           // FIRE MISSILES!
       })
       .setNegativeButton(R.string.cancel, { d, id ->
           // User cancelled the dialog
       })
       .create()

val router = Router.Builder()
       .addRoute(path = "/home", ::showHome)
       .addRoute(path = "/users", ::showUsers)
       .build()
```

To achieve similar behavior with a constructor we would need to introduce special types to hold more data in a single argument:

``` kotlin
val dialog = AlertDialog(context,
   message = R.string.fire_missiles,
   positiveButtonDescription = 
       ButtonDescription(R.string.fire, { d, id ->
           // FIRE MISSILES!
       }),
   negativeButtonDescription = 
       ButtonDescription(R.string.cancel, { d, id ->
           // User cancelled the dialog
       })
)

val router = Router(
   routes = listOf(
       Route("/home", ::showHome),
       Route("/users", ::showUsers)
   )
)
```

This notation is generally badly received in the Kotlin community, and we tend to prefer using DSL (Domain Specific Language) builder for such cases:

``` kotlin
val dialog = context.alert(R.string.fire_missiles) {
   positiveButton(R.string.fire) {
       // FIRE MISSILES!
   }
   negativeButton {
       // User cancelled the dialog
   }
}

val route = router {
   "/home" directsTo ::showHome
   "/users" directsTo ::showUsers
}
```

These kinds of DSL builders are generally preferred over classic builder pattern, since they give more flexibility and cleaner notation. It is true that making a DSL is harder. On the other hand making a builder is already hard. If we decide to invest more time to allow a better notation at the cost of a less obvious definition, why not take this one step further. In return we will have more flexibility and readability. In the next chapter, we are going to talk more about using DSLs for object creation. 

Another advantage of the classic builder pattern is that it can be used as a factory. It might be filled partially and passed further, for example a default dialog in our application: 

``` kotlin
fun Context.makeDefaultDialogBuilder() =
   AlertDialog.Builder(this)
       .setIcon(R.drawable.ic_dialog)
       .setTitle(R.string.dialog_title)
       .setOnCancelListener { it.cancel() }
```

To have a similar possibility in a constructor or factory method, we would need currying, which is not supported in Kotlin. Alternatively we could keep object configuration in a data class and use `copy` to modify an existing one:

``` kotlin
data class DialogConfig(
   val icon: Int = -1,
   val title: Int = -1,
   val onCancelListener: (() -> Unit)? = null
   //...
)

fun makeDefaultDialogConfig() = DialogConfig(
   icon = R.drawable.ic_dialog,
   title = R.string.dialog_title,
   onCancelListener = { it.cancel() }
)
```

Although both options are rarely seen as an option. If we want to define, let’s say, a default dialog for an application, we can create it using a function and pass all customization elements as optional arguments. Such a method would have more control over dialog creation. This is why this advantage of the builder pattern I treat as minor. 

In the end, the builder pattern is rarely the best option in Kotlin. It is sometimes chosen:

- to make code consistent with libraries written in other languages that used builder pattern,
- when we design API to be easily used in other languages that do not support default arguments or DSLs.

Except of that, we rather prefer either a primary constructor with default arguments, or an expressive DSL. 

### Summary

Creating objects using a primary constructor is the most appropriate approach for the vast majority of objects in our projects. Telescoping constructor patterns should be treated as obsolete in Kotlin. I recommend using default values instead, as they are cleaner, more flexible, and more expressive. The builder pattern is very rarely reasonable either as in simpler cases we can just use a primary constructor with named arguments, and when we need to create more complex object we can define a DSL for that.