## Item 48: Eliminate obsolete object references

Programmers that are used to languages with automatic memory management rarely think about freeing objects. In Java, for example, the Garbage Collector (GC) does all the job. Although forgetting about memory management altogether leads to memory leaks - unnecessary memory consumption - and in some cases to `OutOfMemoryError`. The single most important rule is that we should not keep a reference to an object that is not useful anymore. Especially if such an object is big in terms of memory or if there might be a lot of instances of such objects. 

In Android, there is a common beginner’s mistake, where a developer willing to freely access an `Activity` (a concept similar to a window in a desktop application) from any place, stores it in a static or companion object property (classically, in a static field):

``` kotlin
class MainActivity : Activity() {

   override fun onCreate(savedInstanceState: Bundle?) {
       super.onCreate(savedInstanceState)
       //...
       activity = this
   }
  
   //...

   companion object {
       // DON'T DO THIS! It is a huge memory leak
       var activity: MainActivity? = null
   }
}
```

Holding a reference to an activity in a companion object does not let Garbage Collector release it as long as our application is running. Activities are heavy objects, and so it is a huge memory leak. There are some ways to improve it, but it is best not to hold such resources statically at all. **Manage dependencies properly instead of storing them statically.** Also, notice that we might deal with a memory leak when we hold an object storing a reference to another one. Like in the example below, we hold a lambda function that captures a reference to the `MainActivity`:

``` kotlin
class MainActivity : Activity() {

   override fun onCreate(savedInstanceState: Bundle?) {
       super.onCreate(savedInstanceState)
       //...

       // Be careful, we leak a reference to `this`
       logError = { Log.e(this::class.simpleName, 
it.message) }
   }

   //...

   companion object {
       // DON'T DO THIS! A memory leak
       var logError: ((Throwable)->Unit)? = null
   }
}
```

Although problems with memory can be much more subtle. Take a look at the stack implementation below[3](chap65.xhtml#fn-footnote_730_note):

``` kotlin
class Stack {
   private var elements: Array<Any?> =
       arrayOfNulls(DEFAULT_INITIAL_CAPACITY)
   private var size = 0

   fun push(e: Any) {
       ensureCapacity()
       elements[size++] = e
   }

   fun pop(): Any? {
       if (size == 0) {
           throw EmptyStackException()
       }
       return elements[--size]
   }

   private fun ensureCapacity() {
       if (elements.size == size) {
           elements = elements.copyOf(2 * size + 1)
       }
   }

   companion object {
       private const val DEFAULT_INITIAL_CAPACITY = 16
   }
}
```

Can you spot a problem here? Take a minute to think about it.

The problem is that when we pop, we just decrement size, but we don’t free an element on the array. Let’s say that we had 1000 elements on the stack, and we popped nearly all of them one after another and our size is now equal to 1. We should have only one element. We can access only one element. Although our stack still holds 1000 elements and doesn’t allow GC to destroy them. All those objects are wasting our memory. This is why they are called memory leaks. If this leaks accumulate we might face an `OutOfMemoryError`. How can we fix this implementation? A very simple solution is to set `null` in the array when an object is not needed anymore:

``` kotlin
fun pop(): Any? {
   if (size == 0)
       throw EmptyStackException()
   val elem = elements[--size]
   elements[size] = null
   return elem
}
```

This is a rare example and a costly mistake, but there are everyday objects we use that profit, or can profit, from this rule as well. Let’s say that we need a `mutableLazy` property delegate. It should work just like `lazy`, but it should also allow property state mutation. I can define it using the following implementation:

``` kotlin
fun <T> mutableLazy(initializer: () -> T): 
ReadWriteProperty<Any?, T> = MutableLazy(initializer)

private class MutableLazy<T>(
   val initializer: () -> T
) : ReadWriteProperty<Any?, T> {

   private var value: T? = null
   private var initialized = false

   override fun getValue(
       thisRef: Any?,
       property: KProperty<*>
   ): T {
       synchronized(this) {
           if (!initialized) {
               value = initializer()
               initialized = true
           }
           return value as T
       }
   }

   override fun setValue(
       thisRef: Any?,
       property: KProperty<*>,
       value: T
   ) {
       synchronized(this) {
           this.value = value
           initialized = true
       }
   }
}
```

Usage:

``` kotlin
var game: Game? by mutableLazy { readGameFromSave() }

fun setUpActions() {
   startNewGameButton.setOnClickListener {
       game = makeNewGame()
       startGame()
   }
   resumeGameButton.setOnClickListener {
       startGame()
   }
}
```

The above implementation of `mutableLazy` is correct, but has one flaw: `initializer` is not cleaned after usage. It means that it is held as long as the reference to an instance of `MutableLazy` exist even though it is not useful anymore. This is how `MutableLazy` implementation can be improved:

``` kotlin
fun <T> mutableLazy(initializer: () -> T): 
ReadWriteProperty<Any?, T> = MutableLazy(initializer)

private class MutableLazy<T>(
   var initializer: (() -> T)?
) : ReadWriteProperty<Any?, T> {

   private var value: T? = null

   override fun getValue(
       thisRef: Any?,
       property: KProperty<*>
   ): T {
       synchronized(this) {
           val initializer = initializer
           if (initializer != null) {
               value = initializer()
               this.initializer = null
           }
           return value as T
       }
   }

   override fun setValue(
       thisRef: Any?,
       property: KProperty<*>,
       value: T
   ) {
       synchronized(this) {
           this.value = value
           this.initializer = null
       }
   }
}
```

When we set initializer to `null`, the previous value can be recycled by the GC.

How important is this optimization? Not so important to bother for rarely used objects. There is a saying that premature optimization is the root of all evil. Although, it is good to set `null` to unused objects when it doesn’t cost you much to do it. Especially when it is a function type which can capture many variables, or when it is an unknown class like `Any` or a generic type. For example, `Stack` from the above example might be used by someone to hold heavy objects. This is a general purpose tool and we don’t know how it will be used. For such tools, we should care more about optimizations. This is especially true when we are creating a library. For instance, in all 3 implementations of a lazy delegate from Kotlin stdlib, we can see that initializers are set to `null` after usage:

``` kotlin
private class SynchronizedLazyImpl<out T>(
   initializer: () -> T, lock: Any? = null
) : Lazy<T>, Serializable {
   private var initializer: (() -> T)? = initializer
   private var _value: Any? = UNINITIALIZED_VALUE
   private val lock = lock ?: this

   override val value: T
       get() {
           val _v1 = _value
           if (_v1 !== UNINITIALIZED_VALUE) {
               @Suppress("UNCHECKED_CAST")
               return _v1 as T
           }

           return synchronized(lock) {
               val _v2 = _value
               if (_v2 !== UNINITIALIZED_VALUE) {
                   @Suppress("UNCHECKED_CAST") (_v2 as T)
               } else {
                   val typedValue = initializer!!()
                   _value = typedValue
                   initializer = null
                   typedValue
               }
           }
       }

   override fun isInitialized(): Boolean = 
      _value !== UNINITIALIZED_VALUE

   override fun toString(): String = 
      if (isInitialized()) value.toString() 
      else "Lazy value not initialized yet."

   private fun writeReplace(): Any = 
      InitializedLazyImpl(value)
}
```

The general rule is that **when we hold state, we should have memory management in our minds**. Before changing implementation, we should always think of the best trade-off for our project, having in mind not only memory and performance, but also the readability and scalability of our solution. Generally, readable code will also be doing fairly well in terms of performance or memory. Unreadable code is more likely to hide memory leaks or wasted CPU power. Though sometimes those two values stand in opposition and then, in most cases, readability is more important. When we develop a library, performance and memory are often more important.

There are a few common sources of memory leaks we need to discuss. First of all, caches hold objects that might never be used. This is the idea behind caches, but it won’t help us when we suffer from out-of-memory-error. The solution is to use soft references. Objects can still be collected by the GC if memory is needed, but often they will exist and will be used.

Some objects can be referenced using weak reference. For instance a Dialog on a screen. As long as it is displayed, it won’t be Garbage Collected anyway. Once it vanishes we do not need to reference it anyway. It is a perfect candidate for an object being referenced using a weak reference. 

The big problem is that memory leaks are sometimes hard to predict and do not manifest themselves until some point when the application crashes. Which is especially true for Android apps, as there are stricter limits on memory usage than for other kinds of clients, like desktop. This is why we should search for them using special tools. The most basic tool is the heap profiler. We also have some libraries that help in the search for data leaks. For instance, a popular library for Android is LeakCanary which shows a notification whenever a memory leak is detected.

It is important to remember that a need to free objects manually is pretty rare. In most cases, those objects are freed anyway thanks to the scope, or when objects holding them are cleaned. The most important way to avoid cluttering our memory is having variables defined in a local scope (*Item 2: Minimize the scope of variables*) and not storing possibly heavy data in top-level properties or object declarations (including companion objects).