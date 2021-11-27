## Item 31: Define contract with documentation

Think again about the function to display a message from *Item 27: Use abstraction to protect code against changes*:

``` kotlin
fun Context.showMessage(
    message: String, 
    length: MessageLength = MessageLength.LONG
) {
   val toastLength = when(length) {
       SHORT -> Toast.LENGTH_SHORT
       LONG -> Toast.LENGTH_LONG
   }
   Toast.makeText(this, message, toastLength).show()
}

enum class MessageLength { SHORT, LONG }
```

We extracted it to give ourselves the freedom to change how the message is displayed. However, it is not well documented. Another developer might read its code and assume that it always displays a toast. This is the opposite of what we wanted to achieve by naming it in a way not suggesting concrete message type. To make it clear it would be better to add a meaningful KDoc comment explaining what should be expected from this function. 

``` kotlin
/**
* Universal way for the project to display a short 
* message to a user.
* @param message The text that should be shown to
* the user
* @param length How long to display the message.
*/ 
fun Context.showMessage(
    message: String, 
    duration: MessageLength = MessageLength.LONG
) {
   val toastDuration = when(duration) {
       SHORT -> Toast.LENGTH_SHORT
       LONG -> Toast.LENGTH_LONG
   }
   Toast.makeText(this, message, toastDuration).show()
}

enum class MessageLength { SHORT, LONG }
```

In many cases, there are details that are not clearly inferred by the name at all. For instance, *powerset*, even though it is a well-defined mathematical concept, needs an explanation since it is not so well known and interpretation is not clear enough:

``` kotlin
/**
* Powerset returns a set of all subsets of the receiver 
* including itself and the empty set
*/ 
fun <T> Collection<T>.powerset(): Set<Set<T>> =
    if (isEmpty()) setOf(emptySet())
    else take(size - 1)
        .powerset()
        .let { it + it.map { it + last() } }
```

Notice that this description gives us some freedom. It does not specify the order of those elements. As a user, we should not depend on how those elements are ordered. Implementation hidden behind this abstraction can be optimized without changing how this function looks from the outside:

``` kotlin
/**
* Powerset returns a set of all subsets of the receiver 
* including itself and empty set
*/ 
fun <T> Collection<T>.powerset(): Set<Set<T>> = 
      powerset(this, setOf(setOf()))

private tailrec fun <T> powerset(
    left: Collection<T>, 
    acc: Set<Set<T>>
): Set<Set<T>> = when {
   left.isEmpty() -> acc
   else -> {
      val head = left.first()
      val tail = left.drop(1)
      powerset(tail, acc + acc.map { it + head })
   }
}
```

The general problem is that **when the behavior is not documented and the element name is not clear, developers will depend on current implementation instead of on the abstraction we intended to create**. We solve this problem by describing what behavior can be expected. 

### Contract

Whenever we describe some behavior, users treat it as a promise and based on that they adjust their expectations. We call all such expected behaviors a contract of an element. Just like in a real-life contract another side expects us to honor it, here as well users will expect us to keep this contract once it is stable (*Item 28: Specify API stability*).

At this point, defining a contract might sound scary, but actually, it is great for both sides. **When a contract is well specified, creators do not need to worry about how the class is used, and users do not need to worry about how something is implemented under the hood.** Users can rely on this contract without knowing anything about the actual implementation. For creators, the contract gives freedom to change everything as long as the contract is satisfied. **Both users and creators depend on abstractions defined in the contract, and so they can work independently.** Everything will work perfectly fine as long as the contract is respected. This is a comfort and freedom for both sides. 

What if we don’t set a contract? **Without users knowing what they can and cannot do, they’ll depend on implementation details instead. A creator without knowing what users depend on would be either blocked or they would risk breaking users implementations.** As you can see, it is important to specify a contract. 

### Defining a contract

How do we define a contract? There are various ways, including:

- Names - when a name is connected to some more general concept, we expect this element to be consistent with this concept. For instance, when you see `sum` method, you don’t need to read its comment to know how it will behave. It is because the summation is a well defined mathematical concept. 
- Comments and documentation - the most powerful way as it can describe everything that is needed.
- Types - Types say a lot about objects. Each type specifies a set of often well-defined methods, and some types additionally have set-up responsibilities in their documentation. When we see a function, information about return type and arguments types are very meaningful. 

### Do we need comments?

Looking at history, it is amazing to see how opinions in the community fluctuate. When Java was still young, there was a very popular concept of literate programming. It suggested explaining everything in comments[12](chap65.xhtml#fn-footnote_420_note). A decade later we can hear a very strong critique of comments and strong voices that we should omit comments and concentrate on writing readable code instead (I believe that the most influential book was the Clean Code by Robert C. Martin). 

No extreme is healthy. I absolutely agree that we should first concentrate on writing readable code. Though what needs to be understood is that comments before elements (functions or classes) can describe it at a higher level, and set their contract. **Additionally, comments are now often used to automatically generate documentation, which generally is treated as a source of truth in projects.**

Sure, we often do not need comments. For instance, many functions are self-explanatory and they don’t need any special description. We might, for instance, assume that product is a clear mathematical concept that is known by programmers, and leave it without any comment:

``` kotlin
fun List<Int>.product() = fold(1) { acc, i -> acc * i }
```

Obvious comments are a noise that only distracts us. Do not write comments that only describe what is clearly expressed by a function name and parameters. The following example demonstrates an unnecessary comment because the functionality can be inferred from the method’s name and parameter type:

``` kotlin
// Product of all numbers in a list
fun List<Int>.product() = fold(1) { acc, i -> acc * i }
```

I also agree that when we just need to organize our code, instead of comments in the implementation, we should extract a function. Take a look at the example below:

``` kotlin
fun update() {
   // Update users
   for (user in users) {
       user.update()
   }

   // Update books
   for (book in books) {
       updateBook(book)
   }
}
```

Function `update` is clearly composed of extractable parts, and comment suggests that those parts can be described with a different explanation. Therefore it is better to extract those parts into separate abstractions like for instance methods, and their names are clear enough to explain what they mean (just like it in *Item 26: Each function should be written in terms of a single level of abstraction*). 

``` kotlin
fun update() {
   updateUsers()
   updateBooks()
}

private fun updateBooks() {
   for (book in books) {
       updateBook(book)
   }
}

private fun updateUsers() {
   for (user in users) {
       user.update()
   }
}
```

Although comments are often useful and important. To find examples, take a look at nearly any public function from the Kotlin standard library. They have well-defined contracts that give a lot of freedom. For instance, take a look at the function `listOf`: 

``` kotlin
/**
* Returns a new read-only list of given elements. 
* The returned list is serializable (JVM).
* @sample samples.collections.Collections.Lists.
readOnlyList
*/
public fun <T> listOf(vararg elements: T): List<T> = 
     if (elements.size > 0) elements.asList() 
     else emptyList()
```

All it promises is that it returns `List` that is read-only and serializable on JVM. Nothing else. The list does not need to be immutable. No concrete class is promised. This contract is minimalistic, but satisfactory for the needs of most Kotlin developers. You can also see that it points to sample uses, which are also useful when we are learning how to use an element. 

### KDoc format

When we document functions using comments, the official format in which we present that comment is called KDoc. All KDoc comments start with `/**` and end with `*/`, and internally all lines generally start with `*`. Descriptions there are written in KDoc markdown. 

The structure of this KDoc comment is the following:

- The first paragraph of the documentation text is the summary description of the element.
- The second part is the detailed description.
- Every next line begins with a tag. Those tags are used to reference an element to describe it.

Here are tags that are supported:

- @param <name> - Documents a value parameter of a function or a type parameter of a class, property or function. 
- @return - Documents the return value of a function.
- @constructor - Documents the primary constructor of a class.
- @receiver - Documents the receiver of an extension function.
- @property <name> - Documents the property of a class which has the specified name. Used for properties defined on the primary constructor. 
- @throws <class>, @exception <class> - Documents an exception which can be thrown by a method.
- @sample <identifier> - Embeds the body of the function with the specified qualified name into the documentation for the current element, in order to show an example of how the element could be used.
- @see <identifier> - Adds a link to the specified class or method
- @author - Specifies the author of the element being documented.
- @since - Specifies the version of the software in which the element being documented was introduced.
- @suppress - Excludes the element from the generated documentation. Can be used for elements which are not part of the official API of a module but still have to be visible externally.

Both in descriptions and in texts describing tags we can link classes, methods, properties or parameters. Links are in square brackets or with double square brackets when we want to have different description than the name of the linked element.

``` kotlin
/**
* This is an example descriptions linking to [element1], 
* [com.package.SomeClass.element2] and 
* [this element with custom description][element3]
*/
```

All those tags will be understood by Kotlin documentation generation tools. The official one is called Dokka. They generate documentation files that can be published online and presented to outside users. Here is as example documentation with shortened description:

``` kotlin
/**
* Immutable tree data structure.
*
* Class represents immutable tree having from 1 to 
* infinitive number of elements. In the tree we hold 
* elements on each node and nodes can have left and 
* right subtrees...
*
* @param T the type of elements this tree holds.
* @property value the value kept in this node of the tree.
* @property left the left subtree. 
* @property right the right subtree. 
*/
class Tree<T>(
    val value: T, 
    val left: Tree<T>? = null, 
    val right: Tree<T>? = null
) {
    /**
    * Creates a new tree based on the current but with 
    * [element] added.
    * @return newly created tree with additional element. 
    */
    operator fun plus(element: T): Tree { ... }
}
```

Notice that not everything needs to be described. The best documentation is short and on-point describes what might be unclear. 

### Type system and expectations

Type hierarchy is an important source of information about an object. An interface is more than just a list of methods we promise to implement. Classes and interfaces can also have some expectations. If a class promises an expectation, all of its subclasses should guarantee that too. This principle is known as *Liskov substitution principle*, and it is one of the most important rules in the *object-oriented programming*. It is generally translated to “if S is a subtype of T, then objects of type T may be replaced with objects of type S without altering any of the desirable properties of the program”. A simple explanation why it is important is that every class can be used as a superclass, and so if it does not behave as we expect its superclass to behave, we might end up with unexpected failure. In programming, children should always satisfy parents’ contracts. 

One important implication of this rule is that we should properly specify contracts for open functions. For instance, coming back to our car metaphor, we could represent a car in our code using the following interface:

``` kotlin
interface Car {
   fun setWheelPosition(angle: Float)
   fun setBreakPedal(pressure: Double)
   fun setGasPedal(pressure: Double)
}

class GasolineCar: Car {
   // ...
}

class GasCar: Car {
   // ...
}

class ElectricCar: Car {
   // ...
}
```

The problem with this interface is that it leaves a lot of questions. What does `angle` in the `setWheelPosition` function means? In what units it is measured. What if it is not clear for someone what the gas and brake pedals do? People using instances of type `Car` need to know how to use them, and all brands should behave similarly when they are used as a `Car`. We can address those concerns with documentation:

``` kotlin
interface Car {
   /**
    * Changes car direction.
    *
    * @param angle Represents position of wheels in 
    * radians relatively to car axis. 0 means driving
    * straight, pi/2 means driving maximally right, 
    * -pi/2 maximally left. 
    * Value needs to be in (-pi/2, pi/2)
    */
   fun setWheelPosition(angle: Float)

   /**
    * Decelerates vehicle speed until 0.
    *
    * @param pressure The percentage of brake pedal use. 
    * Number from 0 to 1 where 0 means not using break 
    * at all, and 1 means maximal pedal pedal use.
    */
   fun setBreakPedal(pressure: Double)

   /**
    * Accelerates vehicle speed until max speed possible
    * for user.
    *
    * @param pressure The percentage of gas pedal use. 
    * Number from 0 to 1 where 0 means not using gas at 
    * all, and 1 means maximal gas pedal use.
    */
   fun setGasPedal(pressure: Double)
}
```

Now all cars have set a standard that describes how they all should behave. 

Most classes in the stdlib and in popular libraries have well-defined and well-described contracts and expectancies for their children. We should define contracts for our elements as well. Those contracts will make those interfaced truly useful. They will give us the freedom to use classes that implement those interfaces in the way their contract guarantees. 

### Leaking implementation

Implementation details always leak. In a car, different kinds of engines behave a bit differently. We are still able to drive the car, but we can feel a difference. It is fine as this is not described in the contract. 

In programming languages, implementation details leak as well. For instance, calling a function using reflection works, but it is significantly slower than a normal function call (unless it is optimized by the compiler). We will see more examples in the chapter about performance optimization. Though as long as a language works as it promises, everything is fine. We just need to remember and apply good practices. 

In our abstractions, implementation will leak as well, but still, we should protect it as much as we can. We protect it by encapsulation, which can be described as “You can do what I allow, and nothing more”. The more encapsulated classes and functions are, the more freedom we have inside them because we don’t need to think about how one might depend on our implementation.

### Summary

When we define an element, especially parts of external API, we should define a contract. We do that through names, documentation, comments, and types. The contract specifies what the expectations are on those elements. It can also describe how an element should be used. 

A contract gives users confidence about how elements behave now and will behave in the future, and it gives creators the freedom to change what is not specified in the contract. The contract is a kind of agreement, and it works well as long as both sides respect it.