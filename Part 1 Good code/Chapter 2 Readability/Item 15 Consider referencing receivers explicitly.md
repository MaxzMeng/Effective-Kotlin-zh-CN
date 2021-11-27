## Item 15: Consider referencing receivers explicitly

One common place where we might choose a longer structure to make something explicit is when we want to highlight that a function or a property is taken from the receiver instead of being a local or top-level variable. In the most basic situation it means a reference to the class to which the method is associated:

``` kotlin
class User: Person() {
   private var beersDrunk: Int = 0
  
   fun drinkBeers(num: Int) {
       // ...
       this.beersDrunk += num
       // ...
   }
}
```

Similarly, we may explicitly reference an extension receiver (`this` in an extension method) to make its use more explicit. Just compare the following Quicksort implementation written without explicit receivers:

``` kotlin
fun <T : Comparable<T>> List<T>.quickSort(): List<T> {
   if (size < 2) return this
   val pivot = first()
   val (smaller, bigger) = drop(1)
       .partition { it < pivot }
   return smaller.quickSort() + pivot + bigger.quickSort()
}
```

With this one written using them:

``` kotlin
fun <T : Comparable<T>> List<T>.quickSort(): List<T> {
   if (this.size < 2) return this
   val pivot = this.first()
   val (smaller, bigger) = this.drop(1)
       .partition { it < pivot }
   return smaller.quickSort() + pivot + bigger.quickSort()
}
```

The usage is the same for both functions:

``` kotlin
listOf(3, 2, 5, 1, 6).quickSort() // [1, 2, 3, 5, 6]
listOf("C", "D", "A", "B").quickSort() // [A, B, C, D]
```

### Many receivers

Using explicit receivers can be especially helpful when we are in the scope of more than one receiver. We are often in such a situation when we use the `apply`, `with` or `run` functions. Such situations are dangerous and we should avoid them. It is safer to use an object using explicit receiver. To understand this problem, see the following code[2](chap65.xhtml#fn-footnote_20_note):

``` kotlin
class Node(val name: String) {

   fun makeChild(childName: String) =
       create("$name.$childName")
           .apply { print("Created ${name}") }

   fun create(name: String): Node? = Node(name)
} 

fun main() {
   val node = Node("parent")
   node.makeChild("child")
}
```

What is the result? Stop now and spend some time trying to answer yourself before reading the answer. 

It is probably expected that the result should be “Created parent.child”, but the actual result is “Created parent”. Why? To investigate, we can use explicit receiver before `name`:

``` kotlin
class Node(val name: String) {

   fun makeChild(childName: String) =
       create("$name.$childName")
         .apply { print("Created ${this.name}") } 
         // Compilation error

   fun create(name: String): Node? = Node(name)
}         
```

The problem is that the type `this` inside `apply` is `Node?` and so methods cannot be used directly. We would need to unpack it first, for instance by using a safe call. If we do so, result will be finally correct:

``` kotlin
class Node(val name: String) {

   fun makeChild(childName: String) =
       create("$name.$childName")
           .apply { print("Created ${this?.name}") }

   fun create(name: String): Node? = Node(name)
}

fun main() {
    val node = Node("parent")
    node.makeChild("child") 
    // Prints: Created parent.child
}
```

This is an example of bad usage of `apply`. We wouldn’t have such a problem if we used `also` instead, and call `name` on the argument. Using `also` forces us to reference the function’s receiver explicitly the same way as an explicit receiver. In general `also` and `let` are much better choice for additional operations or when we operate on a nullable value. 

``` kotlin
class Node(val name: String) {

   fun makeChild(childName: String) =
       create("$name.$childName")
           .also { print("Created ${it?.name}") }

   fun create(name: String): Node? = Node(name)
}
```

When receiver is not clear, we either prefer to avoid it or we clarify it using explicit receiver. When we use receiver without label, we mean the closest one. When we want to use outer receiver we need to use a label. In such case it is especially useful to use it explicitly. Here is an example showing them both in use:

``` kotlin
class Node(val name: String) {

    fun makeChild(childName: String) =
        create("$name.$childName").apply { 
           print("Created ${this?.name} in "+
               " ${this@Node.name}") 
        }

    fun create(name: String): Node? = Node(name)
}

fun main() {
    val node = Node("parent")
    node.makeChild("child") 
    // Created parent.child in parent
}
```

This way direct receiver clarifies what receiver do we mean. This might be an important information that might not only protect us from errors but also improve readability. 

### DSL marker

There is a context in which we often operate on very nested scopes with different receivers, and we don’t need to use explicit receivers at all. I am talking about Kotlin DSLs. We don’t need to use receivers explicitly because DSLs are designed in such a way. However, in DSLs, it is especially dangerous to accidentally use functions from an outer scope. Think of a simple HTML DSL we use to make an HTML table:

``` kotlin
table {
   tr {
       td { +"Column 1" }
       td { +"Column 2" }
   }
   tr {
       td { +"Value 1" }
       td { +"Value 2" }
   }
}
```

Notice that by default in every scope we can also use methods from receivers from the outer scope. We might use this fact to mess with DSL:

``` kotlin
table {
   tr {
       td { +"Column 1" }
       td { +"Column 2" }
       tr {
           td { +"Value 1" }
           td { +"Value 2" }
      }
   }
}
```

To restrict such usage, we have a special meta-annotation (an annotation for annotations) that restricts implicit usage of outer receivers. This is the `DslMarker`. When we use it on an annotation, and later use this annotation on a class that serves as a builder, inside this builder implicit receiver use won’t be possible. Here is an example of how `DslMarker` might be used:

``` kotlin
@DslMarker
annotation class HtmlDsl

fun table(f: TableDsl.() -> Unit) { /*...*/ }

@HtmlDsl
class TableDsl { /*...*/ }
```

With that, it is prohibited to use outer receiver implicitly:

``` kotlin
table {
   tr {
       td { +"Column 1" }
       td { +"Column 2" }
       tr { // COMPILATION ERROR
           td { +"Value 1" }
           td { +"Value 2" }
       }
   }
}
```

Using functions from an outer receiver requires explicit receiver usage:

``` kotlin
table {
   tr {
       td { +"Column 1" }
       td { +"Column 2" }
       this@table.tr {
           td { +"Value 1" }
           td { +"Value 2" }
       }
   }
}
```

The DSL marker is a very important mechanism that we use to force usage of either the closest receiver or explicit receiver usage. However, it is generally better to not use explicit receivers in DSLs anyway. Respect DSL design and use it accordingly. 

### Summary

Do not change scope receiver just because you can. It might be confusing to have too many receivers all giving us methods we can use. Explicit argument or reference is generally better. When we do change receiver, using explicit receivers improves readability because it clarifies where does the function come from. We can even use a label when there are many receivers to clarify from which one the function comes from. If you want to force using explicit receivers from the outer scope, you can use `DslMarker` meta-annotation.