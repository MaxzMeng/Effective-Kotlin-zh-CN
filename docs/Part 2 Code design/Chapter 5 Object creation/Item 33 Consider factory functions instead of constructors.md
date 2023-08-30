## Item 33: Consider factory functions instead of constructors

The most common way for a class to allow a client to obtain an instance in Kotlin is to provide a primary constructor:

``` kotlin
class MyLinkedList<T>(
   val head: T, 
   val tail: MyLinkedList<T>?
)

val list = MyLinkedList(1, MyLinkedList(2, null))
```

Though constructors are not the only way to create objects. There are many creational design patterns for object instantiation. Most of them revolve around the idea that instead of directly creating an object, a function can create the object for us. For instance, the following top-level function creates an instance of `MyLinkedList`:

``` kotlin
fun <T> myLinkedListOf(
   vararg elements: T
): MyLinkedList<T>? {
   if(elements.isEmpty()) return null
   val head = elements.first()
   val elementsTail = elements
       .copyOfRange(1, elements.size)
   val tail = myLinkedListOf(*elementsTail)
   return MyLinkedList(head, tail)
}

val list = myLinkedListOf(1, 2)
```

Functions used as an alternative to constructors are called factory functions because they produce an object. Using factory functions instead of a constructor has many advantages, including:

- **Unlike constructors, functions have names.** Names explain how an object is created and what the arguments are. For example, let’s say that you see the following code: `ArrayList(3)`. Can you guess what the argument means? Is it supposed to be the first element on the newly created list, or is it the size of the list? It is definitely not self-explanatory. In such situation a name, like `ArrayList.withSize(3)`, would clear up any confusion. Names are really useful: they explain arguments or characteristic ways of object creation. Another reason to have a name is that it solves a conflict between constructors with the same parameter types.
- **Unlike constructors, functions can return an object of any subtype of their return type.** This can be used to provide a better object for different cases. It is especially important when we want to hide actual object implementations behind an interface. Think of `listOf` from stdlib (standard library). Its declared return type is `List` which is an interface. What does it really return? The answer depends on the platform we use. It is different for Kotlin/JVM, Kotlin/JS, and Kotlin/Native because they each use different built-in collections. This is an important optimization made by the Kotlin team. It also gives Kotlin creators much more freedom because the actual type of list might change over time and as long as new objects still implement interface `List` and act the same way, everything will be fine.
- **Unlike constructors, functions are not required to create a new object each time they’re invoked.** It can be helpful because when we create objects using functions, we can include a caching mechanism to optimize object creation or to ensure object reuse for some cases (like in the Singleton pattern). We can also define a static factory function that returns `null` if the object cannot be created. Like `Connections.createOrNull()` which returns `null` when `Connection` cannot be created for some reason.
- **Factory functions can provide objects that might not yet exist.** This is intensively used by creators of libraries that are based on annotation processing. This way, programmers can operate on objects that will be generated or used via proxy without building the project.
- **When we define a factory function outside of an object, we can control its visibility.** For instance, we can make a top-level factory function accessible only in the same file or in the same module.
- **Factory functions can be inline and so their type parameters can be reified.**
- **Factory functions can construct objects which might otherwise be complicated to construct.**
- **A constructor needs to immediately call a constructor of a superclass or a primary constructor.** When we use factory functions, we can postpone constructor usage:

``` kotlin
fun makeListView(config: Config) : ListView {
   val items = … // Here we read items from config
   return ListView(items) // We call actual constructor
}
```

There is a limitation on factory functions usage: it cannot be used in subclass construction. This is because in subclass construction, we need to call the superclass constructor. 

``` kotlin
class IntLinkedList: MyLinkedList<Int>() { 
// Supposing that MyLinkedList is open
  
   constructor(vararg ints: Int): myLinkedListOf(*ints) 
// Error
}
```

That’s generally not a problem, since if we have decided that we want to create a superclass using a factory function, why would we use a constructor for its subclass? We should rather consider implementing a factory function for such class as well.

``` kotlin
class MyLinkedIntList(head: Int, tail: MyLinkedIntList?):
   MyLinkedList<Int>(head, tail)

fun myLinkedIntListOf(vararg elements: Int): 
MyLinkedIntList? {
   if(elements.isEmpty()) return null
   val head = elements.first()
   val elementsTail = elements
       .copyOfRange(1, elements.size)
   val tail = myLinkedIntListOf(*elementsTail)
   return MyLinkedIntList(head, tail)
}
```

The above function is longer than the previous constructor, but it has better characteristics - flexibility, independence of class, and the ability to declare a nullable return type. 

There are strong reasons standing behind factory functions, though what needs to be understood is that **they are not a competition to the primary constructor**[1](chap65.xhtml#fn-constructor). Factory functions still need to use a constructor in their body, so constructor must exist. It can be private if we really want to force creation using factory functions, but we rarely do (*Item 34: Consider primary constructor with named optional arguments*). **Factory functions are mainly a competition to secondary constructors**, and looking at Kotlin projects they generally win as secondary constructors are used rather rarely. **They are also a competition to themselves as there are variety of different kinds of factory functions.** Let’s discuss different Kotlin factory functions:

1. Companion object factory function
2. Extension factory function
3. Top-level factory functions
4. Fake constructors
5. Methods on a factory classes

### Companion Object Factory Function

The most popular way to define a factory function is to define it in a companion object:

``` kotlin
class MyLinkedList<T>(
   val head: T, 
   val tail: MyLinkedList<T>?
) {

   companion object {
      fun <T> of(vararg elements: T): MyLinkedList<T>? {
          /*...*/ 
      }
   }
}

// Usage
val list = MyLinkedList.of(1, 2)
```

Such approach should be very familiar to Java developers because it is a direct equivalent to a static factory method. Though developers of other languages might be familiar with it as well. In some languages, like C++, it is called a *Named Constructor Idiom* as its usage is similar to a constructor, but with a name. 

In Kotlin, this approach works with interfaces too:

``` kotlin
class MyLinkedList<T>(
   val head: T, 
   val tail: MyLinkedList<T>?
): MyList<T> {
   // ...
}

interface MyList<T> {
   // ...
  
   companion object {
       fun <T> of(vararg elements: T): MyList<T>? {
           // ...
       }
   }
}

// Usage
val list = MyList.of(1, 2)
```

Notice that the name of the above function is not really descriptive, and yet it should be understandable for most developers. The reason is that there are some conventions that come from Java and thanks to them, a short word like `of` is enough to understand what the arguments mean. Here are some common names with their descriptions:

- `from` - A type-conversion function that takes a single parameter and returns a corresponding instance of the same type, for example:

``` kotlin
val date: Date = Date.from(instant)
```

- `of` - An aggregation function that takes multiple parameters and returns an instance of the same type that incorporates them, for example:

``` kotlin
val faceCards: Set<Rank> = EnumSet.of(JACK, QUEEN, KING)
```

- `valueOf` - A more verbose alternative to `from` and `of`, for example:

``` kotlin
val prime: BigInteger = BigInteger.valueOf(Integer.MAX_VALUE)
```

- `instance` or `getInstance` - Used in singletons to get the only instance. When parameterized, will return an instance parameterized by arguments. Often we can expect that returned instance to always be the same when arguments are the same, for example:

``` kotlin
val luke: StackWalker = StackWalker.getInstance(options)
```

- `createInstance` or `newInstance` - Like `getInstance`, but this function guarantees that each call returns a new instance, for example:

``` kotlin
val newArray = Array.newInstance(classObject, arrayLen)
```

- `getType` - Like `getInstance`, but used if the factory function is in a different class. Type is the type of object returned by the factory function, for example:

``` kotlin
val fs: FileStore = Files.getFileStore(path)
```

- `newType` - Like `newInstance`, but used if the factory function is in a different class. Type is the type of object returned by the factory function, for example:

``` kotlin
val br: BufferedReader = Files.newBufferedReader(path)
```

Many less-experienced Kotlin developers treat companion object members like static members which need to be grouped in a single block. However, companion objects are actually much more powerful: for example, companion objects can implement interfaces and extend classes. So, we can implement general companion object factory functions like the one below: 

``` kotlin
abstract class ActivityFactory {
   abstract fun getIntent(context: Context): Intent

   fun start(context: Context) {
       val intent = getIntent(context)
       context.startActivity(intent)
   }

   fun startForResult(activity: Activity, requestCode: 
Int) {
       val intent = getIntent(activity)
       activity.startActivityForResult(intent, 
requestCode)
   }
}

class MainActivity : AppCompatActivity() {
   //...

   companion object: ActivityFactory() {
       override fun getIntent(context: Context): Intent =
           Intent(context, MainActivity::class.java)
   }
}

// Usage
val intent = MainActivity.getIntent(context)
MainActivity.start(context)
MainActivity.startForResult(activity, requestCode)
```

Notice that such abstract companion object factories can hold values, and so they can implement caching or support fake creation for testing. The advantages of companion objects are not as well-used as they could be in the Kotlin programming community. Still, if you look at the implementations of the Kotlin team products, you will see that companion objects are strongly used. For instance in the Kotlin Coroutines library, nearly every companion object of coroutine context implements an interface `CoroutineContext.Key` as they all serve as a key we use to identify this context. 

### Extension factory functions

Sometimes we want to create a factory function that acts like an existing companion object function, and we either cannot modify this companion object or we just want to specify a new function in a separate file. In such a case we can use another advantage of companion objects: we can define extension functions for them. 

Suppose that we cannot change the `Tool` interface:

``` kotlin
interface Tool {
   companion object { /*...*/ }
}
```

Nevertheless, we can define an extension function on its companion object:

``` kotlin
fun Tool.Companion.createBigTool( /*...*/ ) : BigTool {
   //... 
}
``` kotlin

At the call site we can then write:

``` kotlin
Tool.createBigTool()
```

This is a powerful possibility that lets us extend external libraries with our own factory methods. One catch is that to make an extension on companion object, there must be some (even empty) companion object:

``` kotlin
interface Tool {
   companion object {}
}
```

### Top-level functions

One popular way to create an object is by using top-level factory functions. Some common examples are `listOf`, `setOf`, and `mapOf`. Similarly, library designers specify top-level functions that are used to create objects. Top-level factory functions are used widely. For example, in Android, we have the tradition of defining a function to create an `Intent` to start an Activity. In Kotlin, the `getIntent()` can be written as a companion object function:

``` kotlin
class MainActivity: Activity {

   companion object {
       fun getIntent(context: Context) = 
           Intent(context, MainActivity::class.java)
   }
}
```

In the Kotlin Anko library, we can use the top-level function `intentFor` with reified type instead:

``` kotlin
intentFor<MainActivity>()
```

This function can be also used to pass arguments:

``` kotlin
intentFor<MainActivity>("page" to 2, "row" to 10)
```

Object creation using top-level functions is a perfect choice for small and commonly created objects like `List` or `Map` because `listOf(1,2,3)` is simpler and more readable than `List.of(1,2,3)`. However, public top-level functions need to be used judiciously. Public top-level functions have a disadvantage: they are available everywhere. It is easy to clutter up the developer’s IDE tips. The problem becomes more serious when top-level functions are named like class methods and they are confused with them. This is why top-level functions should be named wisely.

### Fake constructors

Constructors in Kotlin are used the same way as top-level functions:

``` kotlin
class A
val a = A()
```

They are also referenced the same as top-level functions (and constructor reference implements function interface):

``` kotlin
val reference: ()->A = ::A
```

From a usage point of view, capitalization is the only distinction between constructors and functions. By convention, classes begin with an uppercase letter; functions a lower case letter. Although technically functions can begin with an uppercase. This fact is used in different places, for example, in case of the Kotlin standard library. `List` and `MutableList` are interfaces. They cannot have constructors, but Kotlin developers wanted to allow the following `List`construction:

``` kotlin
List(4) { "User$it" } // [User0, User1, User2, User3]
```

This is why the following functions are included (since Kotlin 1.1) in the Kotlin stdlib:

``` kotlin
public inline fun <T> List(
   size: Int, 
   init: (index: Int) -> T
): List<T> = MutableList(size, init)

public inline fun <T> MutableList(
   size: Int, 
   init: (index: Int) -> T
): MutableList<T> {
   val list = ArrayList<T>(size)
   repeat(size) { index -> list.add(init(index)) }
   return list
}
```

These top-level functions look and act like constructors, but they have all the advantages of factory functions. Lots of developers are unaware of the fact that they are top-level functions under the hood. This is why they are often called *fake constructors*.

Two main reasons why developers choose fake constructors over the real ones are:

- To have “constructor” for an interface
- To have reified type arguments

Except for that, fake constructors should behave like normal constructors. They look like constructors and they should behave this way. If you want to include caching, returning a nullable type or returning a subclass of a class that can be created, consider using a factory function with a name, like a companion object factory method. 

There is one more way to declare a fake constructor. A similar result can be achieved using a companion object with the `invoke` operator. Take a look at the following example:

``` kotlin
class Tree<T> {
  
   companion object {
       operator fun <T> invoke(size: Int, generator: 
(Int)->T): Tree<T>{
           //...
       }
   }
}

// Usage
Tree(10) { "$it" }
```

However, implementing *invoke* in a companion object to make a fake constructor is very rarely used and I do not recommend it. First of all, because it breaks *Item 12: Use operator methods according to their names*. What does it mean to invoke a companion object? Remember that the name can be used instead of the operator: 

``` kotlin
Tree.invoke(10) { "$it" }
```

Invocation is a different operation to object construction. Using the operator in this way is inconsistent with its name. More importantly, this approach is more complicated than just a top-level function. Looking at their reflection shows this complexity. Just compare how reflection looks like when we reference a constructor, fake constructor, and `invoke` function in a companion object:

Constructor:

``` kotlin
val f: ()->Tree = ::Tree
```

Fake constructor:

``` kotlin
val f: ()->Tree = ::Tree
```

Invoke in companion object:

``` kotlin
val f: ()->Tree = Tree.Companion::invoke
```

I recommend using standard top-level functions when you need a fake constructor. These should be used sparingly to suggest typical constructor-like usage when we cannot define a constructor in the class itself, or when we need a capability that constructors do not offer (like a reified type parameter).

### Methods on a factory class

There are many creational patterns associated with factory classes. For instance, abstract factory or prototype. Every one of them has some advantages. 

We will see that some of these approaches are not reasonable in Kotlin. In the next item, we will see that the telescoping constructor and builder pattern rarely make sense in Kotlin. 

Factory classes hold advantages over factory functions because classes can have a state. For instance, this very simple factory class that produces students with next id numbers:

``` kotlin
data class Student(
  val id: Int, 
  val name: String, 
  val surname: String
)

class StudentsFactory {
   var nextId = 0
   fun next(name: String, surname: String) = 
         Student(nextId++, name, surname)
}

val factory = StudentsFactory()
val s1 = factory.next("Marcin", "Moskala")
println(s1) // Student(id=0, name=Marcin, Surname=Moskala)
val s2 = factory.next("Igor", "Wojda")
println(s2) // Student(id=1, name=Igor, Surname=Wojda)
```

Factory classes can have properties and those properties can be used to optimize object creation. When we can hold a state we can introduce different kinds of optimizations or capabilities. We can for instance use caching, or speed up object creation by duplicating previously created objects. 

### Summary

As you can see, Kotlin offers a variety of ways to specify factory functions and they all have their own use. We should have them in mind when we design object creation. Each of them is reasonable for different cases. Some of them should preferably be used with caution: Fake Constructors, Top-Level Factory Method, and Extension Factory Function. The most universal way to define a factory function is by using a Companion Object. It is safe and very intuitive for most developers since usage is very similar to Java Static Factory Methods, and Kotlin mainly inherits its style and practices from Java.