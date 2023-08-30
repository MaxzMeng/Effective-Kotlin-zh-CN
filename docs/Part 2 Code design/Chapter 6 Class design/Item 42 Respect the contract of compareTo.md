## Item 42: Respect the contract of `compareTo`

The `compareTo` method is not in the `Any` class. It is an operator in Kotlin that translates into the mathematical comparison signs:

``` kotlin
obj1 > obj2 // Translates to obj1.compareTo(obj2) > 0
obj1 < obj2 // Translates to obj1.compareTo(obj2) < 0
obj1 >= obj2 // Translates to obj1.compareTo(obj2) >= 0
obj1 <= obj2 // Translates to obj1.compareTo(obj2) <= 0
```

It is also located in the `Comparable<T>` interface. When an object implements this interface or when it has an operator method named `compareTo`, it means that this object has a natural order. Such an order needs to be:

- **Antisymmetric**, meaning that if a >= b and b >= a then a == b. Therefore there is a relation between comparison and equality and they need to be consistent with each other. 
- **Transitive**, meaning that if a >= b and b >= c then a >= c. Similarly when a > b and b > c then a > c. This property is important because without it, sorting of elements might take literally forever in some sorting algorithms. 
- **Connex**, meaning that there must be a relationship between every two elements. So either a >= b, or b >= a. In Kotlin, it is guaranteed by typing system for `compareTo` because it returns `Int`, and every `Int` is either positive, negative or zero. This property is important because if there is no relationship between two elements, we cannot use classic sorting algorithms like quicksort or insertion sort. Instead, we need to use one of the special algorithms for partial orders, like topological sorting. 

### Do we need a compareTo?

In Kotlin we rarely implement `compareTo` ourselves. We get more freedom by specifying the order on a case by case basis than by assuming one global natural order. For instance, we can sort a collection using `sortedBy` and provide a key that is comparable. So in the example below, we sort users by their surname:

``` kotlin
class User(val name: String, val surname: String)
val names = listOf<User>(/*...*/)

val sorted = names.sortedBy { it.surname }
```

What if we need a more complex comparison than just by a key? For that, we can use the `sortedWith` function that sorts elements using a comparator. This comparator can be produced using a function `compareBy`. So in the following example, we first sort users comparing them by their `surname`, and if they match, we compare them by their `name`:

``` kotlin
val sorted = names
    .sortedWith(compareBy({ it.surname }, { it.name }))
```

Surely, we might make `User` implement `Comparable<User>`, but what order should it choose? We might need to sort them by any property. When this is not absolutely clear, it is better to not make such objects comparable. 

`String` has a natural order, which is an alphanumerical order, and so it implements `Comparable<String>`. This fact is very useful because we often do need to sort text alphanumerically. However, it also has its downsides. For instance, we can compare two strings using a comparision sign, which seems highly unintuitive. Most people seeing comparison sign between two strings will be rather confused.

``` kotlin
// DON'T DO THIS! 
print("Kotlin" > "Java") // true
```

Surely there are objects with a clear natural order. Units of measure, date and time are all perfect examples. Although if you are not sure about whether your object has a natural order, it is better to use comparators instead. If you use a few of them often, you can place them in the companion object of your class:

``` kotlin
class User(val name: String, val surname: String) {
   // ...

   companion object {
       val DISPLAY_ORDER =
               compareBy(User::surname, User::name)
   }
}

val sorted = names.sortedWith(User.DISPLAY_ORDER)
```

### Implementing `compareTo`

When we do need to implement `compareTo` ourselves, we have top-level functions that can help us. If all you need is to compare two values, you can use the `compareValues` function:

``` kotlin
class User(
   val name: String, 
   val surname: String
): Comparable<User> {
   override fun compareTo(other: User): Int =
           compareValues(surname, other.surname)
}
```

If you need to use more values, or if you need to compare them using selectors, use `compareValuesBy`:

``` kotlin
class User(
   val name: String, 
   val surname: String
): Comparable<User> {
   override fun compareTo(other: User): Int =
     compareValuesBy(this, other, { it.surname }, { 
it.name })
}
```

This function helps us create most comparators we might need. If you need to implement some with a special logic, remember that it should return:

- 0 if the receiver and `other` are equal
- a positive number if the receiver is greater than `other`
- a negative number if the receiver is smaller than `other`

Once you did that, don’t forget to verify that your comparison is antisymmetric, transitive and connex.