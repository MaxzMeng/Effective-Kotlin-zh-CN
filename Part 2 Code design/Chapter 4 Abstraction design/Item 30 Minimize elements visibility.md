## Item 30: Minimize elements visibility

When we design an API, there are many reasons why we prefer it as lean as possible. Let’s name the most important reasons.

**It is easier to learn and maintain a smaller interface.** Understanding a class is easier when there are a few things we can do with it, than when there are dozens. Maintenance is easier as well. When we make changes, we often need to understand the whole class. When fewer elements are visible, there is less to maintain and to test.

**When we want to make changes, it is way easier to expose something new, rather than to hide an existing element.** All publicly visible elements are part of our public API, and they can be used externally. The longer an element is visible, the more external uses it will have. As such, changing these elements will be harder because they will require updating all usages. Restricting visibility would be even more of a challenge. If you do, you’ll need to carefully consider each usage and provide an alternative. Giving an alternative might not be simple, especially if it was implemented by another developer. Finding out now what were the business requirements might be tough as well. If it is a public library, restricting some elements’ visibility might make some users angry. They will need to adjust their implementation and they will face the same problems - they will need to implement alternative solutions probably years after code was developed. It is much better to force developers to use a smaller API in the first place. 

**A class cannot be responsible for its own state when properties that represent this state can be changed from the outside.** We might have assumptions on a class state that class needs to satisfy. When this state can be directly changed from the outside, the current class cannot guarantee its invariants, because it might be changed externally by someone not knowing about our internal contract. Take a look at `CounterSet` from *Chapter 2*. We correctly restricted the visibility of `elementsAdded` setter. Without it, someone might change it to any value from outside and we wouldn’t be able to trust that this value really represents how many elements were added. Notice that only setter is private. This is a very useful trick. 

``` kotlin
class CounterSet<T>(
       private val innerSet: MutableSet<T> = mutableSetOf\
()
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

For many cases, it is very helpful that all properties are encapsulated by default in Kotlin because we can always restrict the visibility of concrete accessors. 

Protecting internal object state is especially important when we have properties depending on each other. For instance, in the below `mutableLazy` delegate implementation, we expect that if `initialized` is `true`, the `value` is initialized and it contains a value of type `T`. Whatever we do, setter of the `initialized` should not be exposed, because otherwise it cannot be trusted and that can lead to an ugly exception on a different property. 

``` kotlin
class MutableLazyHolder<T>(val initializer: () -> T) {
    private var value: Any? = Any()
    private var initialized = false

    fun get(): T {
        if (!initialized) {
            value = initializer()
            initialized = true
        }
        return value as T
    }

    fun set(value: T) {
        this.value = value
        initialized = true
    }
}
```

**It is easier to track how class changes when they have more restricted visibility.** This makes the property state easier to understand. It is especially important when we are dealing with concurrency. State changes are a problem for parallel programming, and it is better to control and restrict them as much as possible.

### Using visibility modifiers

To achieve a smaller interface from outside, without internal sacrifices, we restrict elements visibility. In general, if there is no reason for an element to be visible, we prefer to have it hidden. This is why if there is no good reason to have less restrictive visibility type, it is a good practice to make the visibility of classes and elements as restrictive as possible. We do that using visibility modifiers. 

For class members, these are 4 visibility modifiers we can use together with their behavior:

- `public` (default) - visible everywhere, for clients who see the declaring class.
- `private` - visible inside this class only.
- `protected` - visible inside this class and in subclasses.
- `internal` - visible inside this module, for clients who see the declaring class.

Top-level elements have 3 visibility modifiers:

- `public` (default) - visible everywhere.
- `private` - visible inside the same file only.
- `internal` - visible inside this module.

Note that the module is not the same as package. In Kotlin it is defined as a set of Kotlin sources compiled together. It might mean:

- a Gradle source set,
- a Maven project,
- an IntelliJ IDEA module,
- a set of files compiled with one invocation of the Ant task.

If your module might be used by another module, change the visibility of your public elements that you don’t want to expose to `internal`. If an element is designed for inheritance and it is only used in a class and subclasses, make it `protected`. If you use element only in the same file or class, make it `private`. This convention is supported by Kotlin as it suggests to restrict visibility to private if an element is used only locally:

![](../../assets/chapter4/chapter4-9.png)

This rule should not be applied to properties in the classes that were designed primarily to hold data (data model classes, DTO). If your server returns a user with an age, and you decided to parse it, you don’t need to hide it just because you don’t use it at the moment. It is there to be used and it is better to have it visible. If you don’t need it, get rid of this property entirely. 

``` kotlin
class User(
       val name: String,
       val surname: String,
       val age: Int
)
```

One big limitation is that when we inherit an API, we cannot restrict the visibility of a member by overriding it. This is because the subclass can always be used as its superclass. This is just another reason to prefer composition instead of inheritance (Item 36: Prefer composition over inheritance).

### Summary

The rule of thumb is that: **Elements visibility should be as restrictive as possible**. Visible elements constitute the public API, and we prefer it as lean as possible because:

- It is easier to learn and maintain a smaller interface. 
- When we want to make changes, it is way easier to expose something than to hide something.
- A class cannot be responsible for its own state when properties that represent this state can be changed from outside.
- It is easier to track how the API changes when they have more restricted visibility.