Kotlin’s type inference is one of the most popular Kotlin features in the JVM world. So popular that Java 10 introduced type inference as well (limited comparing to Kotlin). Though, there are some dangers in using this feature. Above all, we need to remember that the inferred type of an assignment is the exact type of the right side, and not a superclass or interface:

```kotlin
open class Animal
class Zebra: Animal()

fun main() {
   var animal = Zebra()
   animal = Animal() // Error: Type mismatch
}
```

In most cases, this is not a problem. When we have too restrictive type inferred, we just need to specify it and our problem is solved:

``` kotlin
open class Animal
class Zebra: Animal()

fun main() {
   var animal: Animal = Zebra()
   animal = Animal()
}
```

Though, there is no such comfort when we don’t control a library or another module. In such cases, inferred type exposition can be really dangerous. Let’s see an example. 

Let’s say that you have the following interface used to represent car factories:

``` kotlin
1 interface CarFactory {
2    fun produce(): Car
3 }
```

There is also a default car used if nothing else was specified:

``` kotlin
1 val DEFAULT_CAR: Car = Fiat126P()
```

It is produced in most factories, so you made it default. You omitted the type because you decided that `DEFAULT_CAR` is a `Car` anyway:

``` kotlin
1 interface CarFactory {
2    fun produce() = DEFAULT_CAR
3 } 
```

Similarly, someone later looked at `DEFAULT_CAR` and decided that its type can be inferred:

``` kotlin
1 val DEFAULT_CAR = Fiat126P()
```

Now, all your factories can only produce `Fiat126P`. Not good. If you defined this interface yourself, this problem will be probably caught soon and easily fixed. Though, if it is a part of the external API, you might be informed first by angry users.

Except that, the return type is important information when someone does not know API well, and so for the sake of readability, we should make it explicit especially in parts of our API visible from outside (so exposed API). 

#### Summary

The general rule is that if we are not sure about the type, we should specify it. It is important information and we should not hide it (*Item 14: Specify the variable type when it is not clear*). Additionally for the sake of safety, in an external API, we should always specify types. We cannot let them be changed by accident. Inferred types can be too restrictive or can too easily change when our project evolves.