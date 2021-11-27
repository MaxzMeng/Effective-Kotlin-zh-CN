## Item 36: Prefer composition over inheritance


Inheritance is a powerful feature, but it is designed to create a hierarchy of objects with the “is-a” relationship. When such a relationship is not clear, inheritance might be problematic and dangerous. When all we need is a simple code extraction or reuse, inheritance should be used with caution, and we should instead prefer a lighter alternative: class composition. 

### Simple behavior reuse

Let’s start with a simple problem: we have two classes with partially similar behavior - progress bar display before and hide after logic. 

``` kotlin
class ProfileLoader {

  fun load() {
       // show progress
       // load profile
       // hide progress
   }
}

class ImageLoader {

   fun load() {
       // show progress
       // load image
       // hide progress
   }
}
```

From my experience, many developers would extract this common behavior by extracting a common superclass:

``` kotlin
abstract class LoaderWithProgress {

   fun load() {
       // show progress
       innerLoad()
       // hide progress
   }
  
   abstract fun innerLoad()
}

class ProfileLoader: LoaderWithProgress() {

   override fun innerLoad() {
       // load profile
   }
}

class ImageLoader: LoaderWithProgress() {

   override fun innerLoad() {
       // load image
   }
}
```

This approach works for such a simple case, but it has important downsides we should be aware of:

- **We can only extend one class.** Extracting functionalities using inheritance often leads to huge BaseXXX classes that accumulate many functionalities or too deep and complex hierarchies of types.
- **When we extend, we take everything from a class**, which leads to classes that have functionalities and methods they don’t need (a violation of the Interface Segregation Principle).
- **Using superclass functionality is much less explicit**. In general, it is a bad sign when a developer reads a method and needs to jump into superclasses many times to understand how the method works.

Those are strong reasons that should make us think about an alternative, and a very good one is composition. By composition, we mean holding an object as a property (we compose it) and reusing its functionalities. This is an example of how we can use composition instead of inheritance to solve our problem:

``` kotlin
class Progress {
   fun showProgress() { /* show progress */ }
   fun hideProgress() { /* hide progress */ }
}

class ProfileLoader {
   val progress = Progress()
  
   fun load() {
       progress.showProgress()
       // load profile
       progress.hideProgress()
   }
}

class ImageLoader {
   val progress = Progress()

   fun load() {
       progress.showProgress()
       // load image
       progress.hideProgress()
   }
}
```

Notice that composition is harder, as we need to include the composed object and use it in every single class. This is the key reason why many prefer inheritance. However, this additional code is not useless; it informs the reader that progress is used and how it is used. It also gives the developer more power over how progress works.

Another thing to note is that composition is better in a case when we want to extract multiple pieces of functionality. For instance, information that loading has finished:

``` kotlin
class ImageLoader {
   private val progress = Progress()
   private val finishedAlert = FinishedAlert()

   fun load() {
       progress.showProgress()
       // load image
       progress.hideProgress()
       finishedAlert.show()
   }
}
```

We cannot extend more than a single class, so if we would want to use inheritance instead, we would be forced to place both functionalities in a single superclass. This often leads to a complex hierarchy of types used to add these functionalities. Such hierarchies are very hard to read and often also to modify. Just think about what happens if we need alert in two subclasses, but not in the third one? One way to deal with this problem is to use a parameterized constructor:

``` kotlin
abstract class InternetLoader(val showAlert: Boolean) {

   fun load() {
       // show progress
       innerLoad()
       // hide progress
       if (showAlert) {
           // show alert
       }
   }

   abstract fun innerLoad()
}

class ProfileLoader : InternetLoader(showAlert = true) {

   override fun innerLoad() {
       // load profile
   }
}

class ImageLoader : InternetLoader(showAlert = false) {

   override fun innerLoad() {
       // load image
   }
}
```

This is a bad solution. It gives functionality a subclass doesn’t need and blocks it. The problem is compounded when the subclass cannot block other unneeded functionality. When we use inheritance we take everything from the superclass, not only what we need.

### Taking the whole package

When we use inheritance, we take from superclass everything - both methods, expectations (contract) and behavior. Therefore inheritance is a great tool to represent the hierarchy of objects, but not necessarily to just reuse some common parts. For such cases, the composition is better because we can choose what behavior do we need. To think of an example, let’s say that in our system we decided to represent a `Dog` that can `bark` and `sniff`:

``` kotlin
abstract class Dog {
   open fun bark() { /*...*/ }
   open fun sniff() { /*...*/ }
}
```

What if then we need to create a robot dog that can bark but can’t sniff? 

``` kotlin
class Labrador: Dog()

class RobotDog : Dog() {
   override fun sniff() {
       throw Error("Operation not supported")
       // Do you really want that?
   }
}
```

Notice that such a solution violates *interface-segregation principle* - `RobotDog` has a method it doesn’t need. It also violates the Liskov Substitution Principle by breaking superclass behavior. On the other hand, what if your `RobotDog` needs to be a `Robot` class as well because `Robot` can calculate (have `calculate` method)? *Multiple inheritance* is not supported in Kotlin.

``` kotlin
abstract class Robot {
   open fun calculate() { /*...*/ }
}

class RobotDog : Dog(), Robot() // Error
```

These are serious design problems and limitations that do not occur when you use composition instead. When we use composition we choose what we want to reuse. To represent type hierarchy it is safer to use interfaces, and we can implement multiple interfaces. What was not yet shown is that inheritance can lead to unexpected behavior. 

### Inheritance breaks encapsulation

To some degree, when we extend a class, we depend not only on how it works from outside but also on how it is implemented inside. This is why we say that inheritance breaks encapsulation. Let’s look at an example inspired by the book Effective Java by Joshua Bloch. Let’s say that we need a set that will know how many elements were added to it during its lifetime. This set can be created using inheritance from `HashSet`:

``` kotlin
class CounterSet<T>: HashSet<T>() {
   var elementsAdded: Int = 0
       private set

   override fun add(element: T): Boolean {
       elementsAdded++
       return super.add(element)
   }

   override fun addAll(elements: Collection<T>): Boolean {
       elementsAdded += elements.size
       return super.addAll(elements)
   }
}
```

This implementation might look good, but it doesn’t work correctly:

``` kotlin
val counterList = CounterSet<String>()
counterList.addAll(listOf("A", "B", "C"))
print(counterList.elementsAdded) // 6
```

Why is that? The reason is that `HashSet` uses the `add` method under the hood of `addAll`. The counter is then incremented twice for each element added using `addAll`. The problem can be naively solved by removing custom `addAll`function:

``` kotlin
class CounterSet<T>: HashSet<T>() {
   var elementsAdded: Int = 0
       private set

   override fun add(element: T): Boolean {
       elementsAdded++
       return super.add(element)
   }
}
```

Although this solution is dangerous. What if the creators of Java decided to optimize `HashSet.addAll` and implement it in a way that doesn’t depend on the `add` method? If they would do that, this implementation would break with a Java update. Together with this implementation, any other libraries which depend on our current implementation will break as well. The Java creators know this, so they are cautious of making changes to these types of implementations. The same problem affects any library creator or even developers of large projects. How can we solve this problem? We should use composition instead of inheritance:

``` kotlin
class CounterSet<T> {
   private val innerSet = HashSet<T>()
   var elementsAdded: Int = 0
       private set

   fun add(element: T) {
       elementsAdded++
       innerSet.add(element)
   }

   fun addAll(elements: Collection<T>) {
       elementsAdded += elements.size
       innerSet.addAll(elements)
   }
}

val counterList = CounterSet<String>()
counterList.addAll(listOf("A", "B", "C"))
print(counterList.elementsAdded) // 3
```

One problem is that in this case, we lose polymorphic behavior: `CounterSet` is not a `Set` anymore. To keep it, we can use the delegation pattern. The delegation pattern is when our class implements an interface, composes an object that implements the same interface, and forwards methods defined in the interface to this composed object. Such methods are called *forwarding methods*. Take a look at the following example:

``` kotlin
class CounterSet<T> : MutableSet<T> {
   private val innerSet = HashSet<T>()
   var elementsAdded: Int = 0
       private set

   override fun add(element: T): Boolean {
       elementsAdded++
       return innerSet.add(element)
   }

   override fun addAll(elements: Collection<T>): Boolean {
       elementsAdded += elements.size
       return innerSet.addAll(elements)
   }

   override val size: Int
       get() = innerSet.size

   override fun contains(element: T): Boolean =
           innerSet.contains(element)

   override fun containsAll(elements: Collection<T>): 
Boolean = innerSet.containsAll(elements)

   override fun isEmpty(): Boolean = innerSet.isEmpty()

   override fun iterator() =
           innerSet.iterator()

   override fun clear() =
           innerSet.clear()

   override fun remove(element: T): Boolean =
           innerSet.remove(element)

   override fun removeAll(elements: Collection<T>): 
Boolean = innerSet.removeAll(elements)

   override fun retainAll(elements: Collection<T>): 
Boolean = innerSet.retainAll(elements)
}
```

The problem now is that we need to implement a lot of forwarding methods (nine, in this case). Thankfully, Kotlin introduced interface delegation support that is designed to help in this kind of scenario. When we delegate an interface to an object, Kotlin will generate all the required forwarding methods during compilation. Here is Kotlin interface delegation presented in action:

``` kotlin
class CounterSet<T>(
   private val innerSet: MutableSet<T> = mutableSetOf()
) : MutableSet<T> by innerSet {

   var elementsAdded: Int = 0
       private set

   override fun add(element: T): Boolean {
       elementsAdded++
       return innerSet.add(element)
   }

   override fun addAll(elements: Collection<T>): Boolean {
       elementsAdded += elements.size
       return innerSet.addAll(elements)
   }
}
```

This is a case where delegation is a good choice: we need polymorphic behavior and inheritance would be dangerous. In most cases, polymorphic behavior is not needed or we use it in a different way. In such a case composition without delegation is more suitable. It is easier to understand and more flexible.

The fact that inheritance breaks encapsulation is a security concern, but in many cases, the behavior is specified in a contract or we don’t depend on it in subclasses (this is generally true when methods are designed for inheritance). There are other reasons to choose the composition. The composition is easier to reuse and gives us more flexibility. 

### Restricting overriding

To prevent developers from extending classes that are not designed for an inheritance, we can just keep them final. Though if for a reason we need to allow inheritance, still all methods are final by default. To let developers override them, they must be set to open:

``` kotlin
open class Parent {
   fun a() {}
   open fun b() {}
}

class Child: Parent() {
   override fun a() {} // Error
   override fun b() {}
}
```

Use this mechanism wisely and open only those methods that are designed for inheritance. Also remember that when you override a method, you can make it final for all subclasses:

``` kotlin
open class ProfileLoader: InternetLoader() {

   final override fun loadFromInterner() {
       // load profile
   }
}
```

This way you can limit the number of methods that can be overridden in subclasses. 

### Summary

There are a few important differences between composition and inheritance:

- **Composition is more secure** - We do not depend on how a class is implemented, but only on its externally observable behavior. 
- **Composition is more flexible** - We can only extend a single class, while we can compose many. When we inherit, we take everything, while when we compose, we can choose what we need. When we change the behavior of a superclass, we change the behavior of all subclasses. It is hard to change the behavior of only some subclasses. When a class we composed changes, it will only change our behavior if it changed its contract to the outside world.
- **Composition is more explicit** - This is both an advantage and a disadvantage. When we use a method from a superclass we don’t need to reference any receiver (we don’t need to use `this` keyword). It is less explicit, which means that it requires less work but it can be confusing and is more dangerous as it is easy to confuse where a method comes from (is it from the same class, superclass, top-level or is it an extension). When we call a method on a composed object, we know where it comes from. 
- **Composition is more demanding** - We need to use composed object explicitly. When we add some functionalities to a superclass we often do not need to modify subclasses. When we use composition we more often need to adjust usages. 
- **Inheritance gives us a strong polymorphic behavior** - This is also a double-edged sword. From one side, it is comfortable that a dog can be treated like an animal. On the other side, it is very constraining. It must be an animal. Every subclass of the animal should be consistent with animal behavior. Superclass set contract and subclasses should respect it. 

It is a general OOP rule to prefer composition over inheritance, but Kotlin encourages composition even more by making all classes and methods final by default and by making interface delegation a first-class citizen. This makes this rule even more important in Kotlin projects.

When is composition more reasonable then? The rule of thumb: **we should use inheritance when there is a definite “is a” relationship**. Not only linguistically, but meaning that every class that inherits from a superclass needs to “be” its superclass. All unit tests for superclasses should always pass for their subclasses (Liskov substitution principle). Object-oriented frameworks for displaying views are good examples: `Application` in JavaFX, `Activity` in Android, `UIViewController` in iOS, and `React.Component` in React. The same is true when we define our own special kind of view element that always has the same set of functionalities and characteristics. Just remember to design these classes with inheritance in mind, and specify how inheritance should be used. Also, keep methods that are not designed for inheritance final.