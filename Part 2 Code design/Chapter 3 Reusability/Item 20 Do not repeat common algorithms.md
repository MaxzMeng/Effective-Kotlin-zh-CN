## Item 20: Do not repeat common algorithms

I often see developers reimplementing the same algorithms again and again. By algorithms here I mean patterns that are not project-specific, so they do not contain any business logic, and can be extracted into separate modules or even libraries. Those might be mathematical operations, collection processing, or any other common behavior. Sometimes those algorithms can be long and complicated, like optimized sorting algorithms. There are also many simple examples though, like number coercion in a range:

``` kotlin
val percent = when {
   numberFromUser > 100 -> 100
   numberFromUser < 0 -> 0
   else -> numberFromUser
}
```

Notice that we don’t need to implement it because it is already in the stdlib as the `coerceIn` extension function:

``` kotlin
val percent = numberFromUser.coerceIn(0, 100)
```

The advantages of extracting even short but repetitive algorithms are:

- **Programming is faster** because a single call is shorter than an algorithm (list of steps).
- **They are named, so we need to know the concept by name instead of recognizing it by reading its implementation.** This is easier for developers who are familiar with the concept. This might be harder for new developers who are not familiar with these concepts, but it pays off to learn the names of repetitive algorithms. Once they learn it, they will profit from that in the future.
- **We eliminate noise, so it is easier to notice something atypical.** In a long algorithm, it is easy to miss hidden pieces of atypical logic. Think of the difference between `sortedBy` and `sortedByDescending`. Sorting direction is clear when we call those functions, even though their bodies are nearly identical. If we needed to implement this logic every time, it would be easy to confuse whether implemented sorting has natural or descending order. Comments before an algorithm implementation are not really helpful either. Practice shows that developers do change code without updating comments, and over time we lose trust in them. 
- **They can be optimized once, and we profit from this optimization everywhere we use those functions.**

### Learn the standard library

Common algorithms are nearly always already defined by someone else. Most libraries are just collections of common algorithms. The most special among them is the stdlib (standard library). It is a huge collection of utilities, mainly defined as extension functions. Learning the stdlib functions can be demanding, but it is worth it. Without it, developers reinvent the wheel time and time again. To see an example, take a look at this snippet taken from an open-source project:

``` kotlin
override fun saveCallResult(item: SourceResponse) {
   var sourceList = ArrayList<SourceEntity>()
   item.sources.forEach {
       var sourceEntity = SourceEntity()
       sourceEntity.id = it.id
       sourceEntity.category = it.category
       sourceEntity.country = it.country
       sourceEntity.description = it.description
       sourceList.add(sourceEntity)
   }
   db.insertSources(sourceList)
}
```

Using `forEach` here is useless. I see no advantage to using it instead of a for-loop. What I do see in this code though is a mapping from one type to another. We can use the `map` function in such cases. Another thing to note is that the way `SourceEntity` is set-up is far from perfect. This is a JavaBean pattern that is obsolete in Kotlin, and instead, we should use a factory method or a primary constructor (*Chapter 5: Objects creation*). If for a reason someone needs to keep it this way, we should, at least, use `apply` to set up all the properties of a single object implicitly. This is our function after a small clean-up:

``` kotlin
override fun saveCallResult(item: SourceResponse) {
   val sourceEntries = item.sources.map(::sourceToEntry)
   db.insertSources(sourceEntries)
}

private fun sourceToEntry(source: Source) = SourceEntity()
    .apply {
        id = source.id
        category = source.category
        country = source.country
        description = source.description
    }
```

### Implementing your own utils

At some point in every project, we need some algorithms that are not in the standard library. For instance what if we need to calculate a product of numbers in a collection? It is a well-known abstraction, and so it is good to define it as a universal utility function:

``` kotlin
fun Iterable<Int>.product() = 
      fold(1) { acc, i -> acc * i }
```

You don’t need to wait for more than one use. It is a well-known mathematical concept and its name should be clear for developers. Maybe another developer will need to use it later in the future and they’ll be happy to see that it is already defined. Hopefully, that developer will find this function. It is bad practice to have duplicate functions achieving the same results. Each function needs to be tested, remembered and maintained, and so should be considered as a cost. We should be aware not to define functions we don’t need, therefore, we should first search for an existing function before implementing our own.

Notice that `product`, just like most functions in the Kotlin stdlib, is an extension function. There are many ways we can extract common algorithms, starting from top-level functions, property delegates and ending up on classes. Although extension functions are a really good choice:

- Functions do not hold state, and so they are perfect to represent behavior. Especially if it has no side-effects.
- Compared to top-level functions, extension functions are better because they are suggested only on objects with concrete types. 
- It is more intuitive to modify an extension receiver than an argument. 
- Compared to methods on objects, extensions are easier to find among hints since they are suggested on objects. For instance `"Text".isEmpty()` is easier to find than `TextUtils.isEmpty("Text")`. It is because when you place a dot after `"Text"` you’ll see as suggestions all extension functions that can be applied to this object. To find `TextUtils.isEmpty` you would need to guess where is it stored, and you might need to search through alternative util objects from different libraries. 
- When we are calling a method, it is easy to confuse a top-level function with a method from the class or superclass, and their expected behavior is very different. Top-level extension functions do not have this problem, because they need to be invoked on an object. 

### Summary

Do not repeat common algorithms. First, it is likely that there is a stdlib function that you can use instead. This is why it is good to learn the standard library. If you need a known algorithm that is not in the stdlib, or if you need a certain algorithm often, feel free to define it in your project. A good choice is to implement it as an extension function.