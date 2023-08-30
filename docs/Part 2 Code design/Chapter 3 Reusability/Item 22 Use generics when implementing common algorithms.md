## Item 22: Use generics when implementing common algorithms

Similarly, as we can pass a value to a function as an argument, we can pass a type as a type argument. Functions that accept type arguments (so having type parameters) are called generic functions. One known example is the `filter` function from stdlib that has type parameter `T`:

``` kotlin
inline fun <T> Iterable<T>.filter(
    predicate: (T) -> Boolean
): List<T> {
    val destination = ArrayList<T>()
    for (element in this) { 
        if (predicate(element)) {
           destination.add(element)
        }
    }
    return destination   
}
```

Type parameters are useful to the compiler since they allow it to check and correctly infer types a bit further, what makes our programs safer and programming more pleasurable for developers. For instance, when we use `filter`, inside the lambda expression, the compiler knows that an argument is of the same type as the type of elements in the collection, so it protects us from using something illegal and the IDE can give us useful suggestions. 

![](../../assets/chapter1/chapter1-3.png)

Generics were primarily introduced to classes and interfaces to allow the creation of collections with only concrete types, like `List<String>` or `Set<User>`. Those types are lost during compilation but when we are developing, the compiler forces us to pass only elements of the correct type. For instance `Int` when we add to `MutableList<Int>`. Also, thanks to them, the compiler knows that the returned type is `User` when we get an element from `Set<User>`. This way type parameters help us a lot in statically-typed languages. Kotlin has powerful support for generics that is not well understood and from my experience even experienced Kotlin developers have gaps in their knowledge especially about variance modifiers. So let’s discuss the most important aspects of Kotlin generics in this and in *Item 24: Consider variance for generic types*. 

### Generic constraints

One important feature of type parameters is that they can be constrained to be a subtype of a concrete type. We set a constraint by placing supertype after a colon. This type can include previous type parameters:

``` kotlin
fun <T : Comparable<T>> Iterable<T>.sorted(): List<T> { 
   /*...*/ 
}

fun <T, C : MutableCollection<in T>>
Iterable<T>.toCollection(destination: C): C {
   /*...*/
}

class ListAdapter<T: ItemAdaper>(/*...*/) { /*...*/ }
```

One important result of having a constraint is that instances of this type can use all the methods this type offers. This way when `T` is constrained as a subtype of `Iterable<Int>`, we know that we can iterate over an instance of type `T`, and that elements returned by the iterator will be of type `Int`. When we constraint to `Comparable<T>`, we know that this type can be compared with itself. Another popular choice for a constraint is `Any` which means that a type can be any non-nullable type:

``` kotlin
inline fun <T, R : Any> Iterable<T>.mapNotNull(
   transform: (T) -> R?
): List<R> {
   return mapNotNullTo(ArrayList<R>(), transform)
}
```

In rare cases in which we might need to set more than one upper bound, we can use `where` to set more constraints:

``` kotlin
fun <T: Animal> pet(animal: T) where T: GoodTempered { 
   /*...*/ 
}

// OR

fun <T> pet(animal: T) where T: Animal, T: GoodTempered { 
   /*...*/ 
}
```

### Summary

Type parameters are an important part of Kotlin typing system. We use them to have type-safe generic algorithms or generic objects. Type parameters can be constrained to be a subtype of a concrete type. When they are, we can safely use methods offered by this type.