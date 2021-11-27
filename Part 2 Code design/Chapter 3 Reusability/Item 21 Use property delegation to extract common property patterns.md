## Item 21: Use property delegation to extract common property patterns

One new feature introduced by Kotlin that supports code reuse is property delegation. It gives us a universal way to extract common property behavior. One important example is the lazy property - a property that needs to be initialized on demand during its first use. Such a pattern is really popular, and in languages that do not support its extraction (like Java or JavaScript), it is implemented every time it is needed. In Kotlin, it can easily be extracted using property delegation. In the Kotlin stdlib, we can find the function `lazy` that returns a property delegate that implements the lazy property pattern:

``` kotlin
val value by lazy { createValue() }
```

This is not the only repetitive property pattern. Another important example is the observable property - a property that does something whenever it is changed. For instance, let’s say that you have a list adapter drawing a list. Whenever data change inside it, we need to redraw changed items. Or you might need to log all changes of a property. Both cases can be implemented using `observable` from stdlib:

``` kotlin
var items: List<Item> by 
   Delegates.observable(listOf()) { _, _, _ ->
       notifyDataSetChanged()
   }

var key: String? by 
   Delegates.observable(null) { _, old, new ->
       Log.e("key changed from $old to $new")
   }
```

The lazy and observable delegates are not special from the language’s point of view. They can be extracted thanks to a more general property delegation mechanism that can be used to extract many other patterns as well. Some good examples are View and Resource Binding, Dependency Injection (formally Service Location), or Data Binding. Many of these patterns require annotation processing in Java, however Kotlin allows you to replace them with easy, type-safe property delegation.

``` kotlin
// View and resource binding example in Android
private val button: Button by bindView(R.id.button)
private val textSize by bindDimension(R.dimen.font_size)
private val doctor: Doctor by argExtra(DOCTOR_ARG)

// Dependency Injection using Koin 
private val presenter: MainPresenter by inject()
private val repository: NetworkRepository by inject()
private val vm: MainViewModel by viewModel()

// Data binding
private val port by bindConfiguration("port")
private val token: String by preferences.bind(TOKEN_KEY)
```

To understand how this is possible and how we can extract other common behavior using property delegation, let’s start from a very simple property delegate. Let’s say that we need to track how some properties are used, and for that, we define custom getters and setters that log their changes:

``` kotlin
var token: String? = null
   get() {
       print("token returned value $field")
       return field
   }
   set(value) {
       print("token changed from $field to $value")
       field = value
   }

var attempts: Int = 0
   get() {
       print("attempts returned value $field")
       return field
   }
   set(value) {
       print("attempts changed from $field to $value")
       field = value
   }
```

Even though their types are different, the behavior of those two properties is nearly identical. It seems like a repeatable pattern that might be needed more often in our project. This behavior can be extracted using property delegation. Delegation is based on the idea that a property is defined by its accessors - the getter in a `val`, and the getter and setter in a `var` - and those methods can be delegated into methods of another object. The getter will be delegated to the `getValue` function, and the setter to the `setValue` function. We then place such an object on the right side of the `by` keyword. To keep exactly the same property behaviors as in the example above, we can create the following delegate:

``` kotlin
var token: String? by LoggingProperty(null)
var attempts: Int by LoggingProperty(0)

private class LoggingProperty<T>(var value: T) {
   operator fun getValue(
       thisRef: Any?, 
       prop: KProperty<*>
   ): T {
       print("${prop.name} returned value $value")
       return value
   }

    operator fun setValue(
        thisRef: Any?, 
        prop: KProperty<*>, 
        newValue: T
    ) {
        val name = prop.name
        print("$name changed from $value to $newValue")
        value = newValue
    }
}
```

To fully understand how property delegation works, take a look at what `by` is compiled to. The above `token` property will be compiled to something similar to the following code[6](chap65.xhtml#fn-footnote_3211_note):

``` kotlin
@JvmField
private val `token$delegate` = 
    LoggingProperty<String?>(null)
var token: String?
   get() = `token$delegate`.getValue(this, ::token)
   set(value) {
       `token$delegate`.setValue(this, ::token, value)
   }
```

As you can see, `getValue` and `setValue` methods operate not only on the value, but they also receive a bounded reference to the property, as well as a context (`this`). The reference to the property is most often used to get its name, and sometimes to get information about annotations. Context gives us information about where the function is used. 

When we have multiple `getValue` and `setValue` methods but with different context types, different ones will be chosen in different situations. This fact can be used in clever ways. For instance, we might need a delegate that can be used in different kinds of views, but with each of them, it should behave differently based on what is offered by the context:

``` kotlin
class SwipeRefreshBinderDelegate(val id: Int) {
    private var cache: SwipeRefreshLayout? = null

    operator fun getValue(
       activity: Activity, 
       prop: KProperty<*>
    ): SwipeRefreshLayout {
       return cache ?: activity
           .findViewById<SwipeRefreshLayout>(id)
           .also { cache = it }
    }

    operator fun getValue(
       fragment: Fragment, 
       prop: KProperty<*>
    ): SwipeRefreshLayout {
       return cache ?: fragment.view
           .findViewById<SwipeRefreshLayout>(id)
           .also { cache = it }
    }
}
```

To make it possible to use an object as a property delegate, all it needs is `getValue` operator for `val`, and `getValue` and `setValue` operators for `var`. This operation can be a member function, but it can be also an extension. For instance, `Map<String, *>` can be used as a property delegate:

``` kotlin
val map: Map<String, Any> = mapOf(
   "name" to "Marcin",
   "kotlinProgrammer" to true
)
val name by map
print(name) // Marcin
```

It is possible thanks to the fact that there is the following extension function in the Kotlin stdlib:

``` kotlin
inline operator fun <V, V1 : V> Map<in String, V>
.getValue(thisRef: Any?, property: KProperty<*>): V1 = 
getOrImplicitDefault(property.name) as V1
```

Kotlin standard library has some property delegates that we should know. Those are:

- `lazy`
- `Delegates.observable`
- `Delegates.vetoable`
- `Delegates.notNull`

It is worth to know them, and if you notice a common pattern around properties in your project, remember that you can make your own property delegate. 

### Summary

Property delegates have full control over properties and have nearly all the information about their context. This feature can be used to extract practically any property behavior. `lazy` and `observable` are just two examples from the standard library. Property delegation is a universal way to extract property patterns, and as practice shows, there are a variety of such patterns. It is a powerful and generic feature that every Kotlin developer should have in their toolbox. When we know it, we have more options for common pattern extraction or to define a better API.