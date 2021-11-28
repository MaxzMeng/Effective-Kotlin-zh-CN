## Item 52: Consider using mutable collections

The biggest advantage of using mutable collections instead of immutable is that they are faster in terms of performance. When we add an element to an immutable collection, we need to create a new collection and add all elements to it. Here is how it is currently implemented in the Kotlin stdlib (Kotlin 1.2):

``` kotlin
operator fun <T> Iterable<T>.plus(element: T): List<T> {
   if (this is Collection) return this.plus(element)
   val result = ArrayList<T>()
   result.addAll(this)
   result.add(element)
   return result
}
```

Adding all elements from a previous collection is a costly process when we deal with bigger collections. This is why using mutable collections, especially if we need to add elements, is a performance optimization. On the other hand *Item 1: Limit mutability* taught us the advantages of using immutable collections for safety. Although notice that those arguments rarely apply to local variables where synchronization or encapsulation is rarely needed. This is why for local processing, it generally makes more sense to use mutable collections. This fact can be reflected in the standard library where all collection processing functions are internally implemented using mutable collections:

``` kotlin
inline fun <T, R> Iterable<T>.map(
    transform: (T) -> R
): List<R> {
   val size = if (this is Collection<*>) this.size else 10
   val destination = ArrayList<R>(size)
   for (item in this)
       destination.add(transform(item))
   return destination
}
```

Instead of using immutable collections:

``` kotlin
// This is not how map is implemented
inline fun <T, R> Iterable<T>.map(
    transform: (T) -> R
): List<R> {
   var destination = listOf<R>()
   for (item in this)
       destination += transform(item)
   return destination
}
```

### Summary

Adding to mutable collections is generally faster, but immutable collections give us more control over how they are changed. Though in local scope we generally do not need that control, so mutable collections should be preferred. Especially in utils, where element insertion might happen many times.