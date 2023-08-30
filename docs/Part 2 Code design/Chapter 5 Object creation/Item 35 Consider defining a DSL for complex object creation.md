## Item 35: Consider defining a DSL for complex object creation

A set of Kotlin features used together allows us to make a configuration-like Domain Specific Language (DSL). Such DSLs are useful when we need to define more complex objects or a hierarchical structure of objects. They are not easy to define, but once they are defined they hide boilerplate and complexity, and a developer can express his or her intentions clearly.

For instance, Kotlin DSL is a popular way to express HTML: both classic HTML, and React HTML. This is how it can look like:

``` kotlin
body {
   div {
       a("https://kotlinlang.org") {
           target = ATarget.blank
           +"Main site"
       }
   }
   +"Some content"
}
```

![View from the above HTML DSL](../../assets/chapter5/chapter5-1.png)

Views on other platforms can be defined using DSLs as well. Here is a simple Android view defined using the Anko library:

``` kotlin
verticalLayout {
   val name = editText()
   button("Say Hello") {
       onClick { toast("Hello, ${name.text}!") }
   }
}
```

![View from the above Android View DSL](../../assets/chapter5/chapter5-2.png)

Similarly with desktop applications. Here is a view defined on TornadoFX that is built on top of the JavaFX:

``` kotlin
class HelloWorld : View() {
   override val root = hbox {
       label("Hello world") {
           addClass(heading)
       }

       textfield {
           promptText = "Enter your name"
       }
   }
}
```

![View from the above TornadoFX DSL](../../assets/chapter5/chapter5-2.png)

DSLs are also often used to define data or configurations. Here is API definition in Ktor, also a DSL:

``` kotlin
fun Routing.api() {
   route("news") {
       get {
           val newsData = NewsUseCase.getAcceptedNews()
           call.respond(newsData)
       }
       get("propositions") {
           requireSecret()
           val newsData = NewsUseCase.getPropositions()
           call.respond(newsData)
       }
   }
   // ...
}
```

And here are test case specifications defined in Kotlin Test:

``` kotlin
class MyTests : StringSpec({
   "length should return size of string" {
       "hello".length shouldBe 5
   }
   "startsWith should test for a prefix" {
       "world" should startWith("wor")
   }
})
```

We can even use Gradle DSL to define Gradle configuration:

``` groovy
plugins {
   `java-library`
}

dependencies { 
   api("junit:junit:4.12")
   implementation("junit:junit:4.12")
   testImplementation("junit:junit:4.12")
}

configurations {
   implementation {
       resolutionStrategy.failOnVersionConflict()
   }
}

sourceSets { 
   main {
       java.srcDir("src/core/java")
   }
}

java {
   sourceCompatibility = JavaVersion.VERSION_11
   targetCompatibility = JavaVersion.VERSION_11
}

tasks {
   test {
       testLogging.showExceptions = true
   }
}
```

Creating complex and hierarchical data structures become easier with DSLs. Inside those DSLs we can use everything that Kotlin offers, and we have useful hints as DSLs in Kotlin are fully type-safe (unlike Groovy). It is likely that you already used some Kotlin DSL, but it is also important to know how to define them yourself.

### Defining your own DSL

To understand how to make own DSLs, it is important to understand the notion of function types with a receiver. But before that, we’ll first briefly review the notion of function types themselves. The function type is a type that represents an object that can be used as a function. For instance, in the `filter` function, it is there to represent a predicate that decides if an element can be accepted or not.

``` kotlin
inline fun <T> Iterable<T>.filter(
   predicate: (T) -> Boolean
): List<T> {
   val list = arrayListOf<T>()
   for (elem in this) {
       if (predicate(elem)) {
           list.add(elem)
       }
   }
   return list
}
```

Here are a few examples of function types:

- `()->Unit` - Function with no arguments and returns `Unit`. 
- `(Int)->Unit` - Function that takes `Int` and returns `Unit`. 
- `(Int)->Int` - Function that takes `Int` and returns `Int`. 
- `(Int, Int)->Int` - Function that takes two arguments of type `Int` and returns `Int`. 
- `(Int)->()->Unit -` Function that takes `Int` and returns another function. This other function has no arguments and returns `Unit`. 
- `(()->Unit)->Unit` - Function that takes another function and returns `Unit`. This other function has no arguments and returns `Unit`. 

The basic ways to create instances of function types are:

- Using lambda expressions
- Using anonymous functions
- Using function references

For instance, think about the following function:

``` kotlin
fun plus(a: Int, b: Int) = a + b
```

Analogous functions can be created in the following ways:

```
val plus1: (Int, Int)->Int = { a, b -> a + b }
val plus2: (Int, Int)->Int = fun(a, b) = a + b
val plus3: (Int, Int)->Int = ::plus
```

In the above example, property types are specified and so argument types in the lambda expression and in the anonymous function can be inferred. It could be the other way around. If we specify argument types, then the function type can be inferred.

```
val plus4 = { a: Int, b: Int -> a + b }
val plus5 = fun(a: Int, b: Int) = a + b
```

Function types are there to represent objects that represent functions. An anonymous function even looks the same as a normal function, but without a name. A lambda expression is a shorter notation for an anonymous function. 

Although if we have function types to represent functions, what about extension functions? Can we express them as well? 

``` kotlin
fun Int.myPlus(other: Int) = this + other
```

It was mentioned before that we create an anonymous function in the same way as a normal function but without a name. And so anonymous extension functions are defined the same way:

``` kotlin
val myPlus = fun Int.(other: Int) = this + other
```

What type does it have? The answer is that there is a special type to represent extension functions. It is called *function type with receiver*. It looks similar to a normal function type, but it additionally specifies the receiver type before its arguments and they are separated using a dot:

``` kotlin
val myPlus: Int.(Int)->Int = 
    fun Int.(other: Int) = this + other
```

Such a function can be defined using a lambda expression, specifically a lambda expression with receiver, since inside its scope the `this` keyword references the extension receiver (an instance of type `Int` in this case):

``` kotlin
val myPlus: Int.(Int)->Int = { this + it }
```

Object created using anonymous extension function or lambda expression with receiver can be invoked in 3 ways:

- Like a standard object, using `invoke` method. 
- Like a non-extension function. 
- Same as a normal extension function. 

``` kotlin
myPlus.invoke(1, 2)
myPlus(1, 2)
1.myPlus(2)
```

The most important trait of the function type with receiver is that it changes what `this` refers to. `this` is used for instance in the `apply` function to make it easier to reference the receiver object’s methods and properties:

``` kotlin
inline fun <T> T.apply(block: T.() -> Unit): T {
   this.block()
   return this
}

class User {
   var name: String = ""
   var surname: String = ""
}

val user = User().apply {
   name = "Marcin"
   surname = "Moskała"
}
```

Function type with a receiver is the most basic building block of Kotlin DSL. Let’s create a very simple DSL that would allow us to make the following HTML table:

``` kotlin
fun createTable(): TableBuilder = table {
   tr {
       for (i in 1..2) {
           td {
               +"This is column $i"
           }
       }
   }
}
```

Starting from the beginning of this DSL, we can see a function `table`. We are at top-level without any receivers so it needs to be a top-level function. Although inside its function argument you can see that we use `tr`. The `tr` function should be allowed only inside the table definition. This is why the `table` function argument should have a receiver with such a function. Similarly, the `tr` function argument needs to have a receiver that will contain a `td` function.

``` kotlin
fun table(init: TableBuilder.()->Unit): TableBuilder {
   //...
}

class TableBuilder {
   fun tr(init: TrBuilder.() -> Unit) { /*...*/ }
}

class TrBuilder {
   fun td(init: TdBuilder.()->Unit) { /*...*/ }
}

class TdBuilder
```

How about this statement:

```
+"This is row $i"
```

What is that? This is nothing else but a unary plus operator on String. It needs to be defined inside TdBuilder:

``` kotlin
class TdBuilder {
   var text = ""
  
   operator fun String.unaryPlus() {
       text += this
   }
}
```

Now our DSL is well defined. To make it work fine, at every step we need to create a builder and initialize it using a function from parameter (`init` in the example below). After that, the builder will contain all the data specified in this `init`function argument. This is the data we need. Therefore we can either return this builder or we can produce another object holding this data. In this example, we’ll just return builder. This is how the `table` function could be defined:

``` kotlin
fun table(init: TableBuilder.()->Unit): TableBuilder {
   val tableBuilder = TableBuilder()
   init.invoke(tableBuilder)
   return tableBuilder
}
```

Notice that we can use the `apply` function, as shown before, to shorten this function: 

``` kotlin
fun table(init: TableBuilder.()->Unit) = 
     TableBuilder().apply(init)
```

Similarly we can use it in other parts of this DSL to be more concise:

``` kotlin
class TableBuilder {
   var trs = listOf<TrBuilder>()

   fun tr(init: TrBuilder.()->Unit) {
       trs = trs + TrBuilder().apply(init)
   }
}

class TrBuilder {
   var tds = listOf<TdBuilder>()

   fun td(init: TdBuilder.()->Unit) {
       tds = tds + TdBuilder().apply(init)
   }
}
```

This is a fully functional DSL builder for HTML table creation. It could be improved using a `DslMarker` explained in Item 15: Consider referencing receiver explicitly. 

### When should we use it?

DSLs give us a way to define information. It can be used to express any kind of information you want, but it is never clear to users how this information will be later used. In Anko, TornadoFX or HTML DSL we trust that the view will be correctly built based on our definitions, but it is often hard to track how exactly. Some more complicated uses can be hard to discover. Usage can be also confusing to those not used to them. Not to mention maintenance. The way how they are defined can be a cost - both in developer confusion and in performance. DSLs are an overkill when we can use other, simpler features instead. Though they are really useful when we need to express:

- complicated data structures,
- hierarchical structures,
- huge amount of data.

Everything can be expressed without DSL-like structure, by using builders or just constructors instead. **DSLs are about boilerplate elimination for such structures.** You should consider using DSL when you see repeatable boilerplate code[4](chap65.xhtml#fn-boilerplate)and there are no simpler Kotlin features that can help. 

### Summary

A DSL is a special language inside of a language. It can make it really simple to create complex object, and even whole object hierarchies, like HTML code or complex configuration files. On the other hand DSL implementations might be confusing or hard for new developers. They are also hard to define. This is why they should be only used when they offer real value. For instance, for the creation of a really complex object, or possibly for complex object hierarchies. This is why they are also preferably defined in libraries rather than in projects. It is not easy to make a good DSL, but a well defined DSL can make our project much better.