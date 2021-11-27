## Item 23: Avoid shadowing type parameters

It is possible to define property and parameters with the same name due to shadowing. Local parameter shadows outer scope property. There is no warning because such a situation is not uncommon and is rather visible for developers:

``` kotlin
class Forest(val name: String) {
  
   fun addTree(name: String) {
       // ...
   }
}
```

On the other hand, the same can happen when we shadow class type parameter with a function type parameter. Such a situation is less visible and can lead to serious problems. This mistake is often done by developers not understanding well how generics work. 

``` kotlin
interface Tree
class Birch: Tree
class Spruce: Tree

class Forest<T: Tree> {

   fun <T: Tree> addTree(tree: T) {
       // ...
   }
}
```

The problem is that now `Forest` and `addTree` type parameters are independent of each other:

``` kotlin
val forest = Forest<Birch>()
forest.addTree(Birch())
forest.addTree(Spruce())
```

Such situation is rarely desired and might be confusing. One solution is that `addTree` should use the class type parameter `T`:

``` kotlin
class Forest<T: Tree> {

   fun addTree(tree: T) {
       // ...
   }
}

// Usage
val forest = Forest<Birch>()
forest.addTree(Birch())
forest.addTree(Spruce()) // ERROR, type mismatch
```

If we need to introduce a new type parameter, it is better to name it differently. Note that it can be constrained to be a subtype of the other type parameter:

``` kotlin
class Forest<T: Tree> {

   fun <ST: T> addTree(tree: ST) {
       // ...
   }
}
```

### Summary

Avoid shadowing type parameters, and be careful when you see that type parameter is shadowed. Unlike for other kinds of parameters, it is not intuitive and might be highly confusing.