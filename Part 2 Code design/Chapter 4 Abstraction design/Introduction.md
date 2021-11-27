# Chapter 4: Abstraction design

Abstraction is one of the most important concepts of the programming world. In OOP (Object-Oriented Programming) abstraction is one of three core concepts (along with encapsulation and inheritance). In the functional programming community, it is common to say that all we do in programming is abstraction and composition. As you can see, we treat abstraction seriously. Although what is an abstraction? The definition I find most useful comes from Wikipedia:

> Abstraction is a process or result of generalization, removal of properties, or distancing of ideas from objects. 
> https://en.wikipedia.org/wiki/Abstraction_(disambiguation)

In other words, by abstraction we mean a form of simplification used to hide complexity. A fundamental example in programming is the interface. It is an abstraction of a class because it expresses only a subset of traits. Concretely, it is a set of methods and properties. 

![](../../assets/chapter4/chapter4-1.png)

There is no single abstraction for every instance. There are many. In terms of objects, it can be expressed by many interfaces implemented or by multiple superclasses. It is an inseparable feature of abstraction that it decides what should be hidden and what should be exposed. 

![](../../assets/chapter4/chapter4-2.png)

### Abstraction in programming

We often forget how abstract everything we do in programming is. When we type a number, it is easy to forget that it is actually represented by zeros and ones. When we type some String, it is easy to forget that it is a complex object where each character is represented on a defined charset, like UTF-8. 

Designing abstractions is not only about separating modules or libraries. Whenever you define a function, you hide its implementation behind this function’s signature. This is an abstraction! 

Let’s do a thought experiment: what if it wasn’t possible to define a method `maxOf` that would return the biggest of two numbers:

``` kotlin
fun maxOf(a: Int, b: Int) = if (a > b) a else b
```

Of course, we could get along without ever defining this function, by always writing the full expression and never mentioning `maxOf` explicitly: 

``` kotlin
val biggest = if (x > y) x else y

val height = 
    if (minHeight > calculatedHeight) minHeight 
    else calculatedHeight
```

However, this would place us at a serious disadvantage. It would force us to always work at the level of the particular operations that happen to be primitives in the language (comparison, in this case) rather than in terms of higher-level operations. Our programs would be able to compute which number is bigger, but our language would lack the ability to express the concept of choosing a bigger number. This problem is not abstract at all. Until version 8, Java lacked the capability to easily express mapping on a list. Instead, we had to use repeatable structures to express this concept:

``` kotlin
// Java
List<String> names = new ArrayList<>();
for (User user : users) {
   names.add(user.getName());
}
```

In Kotlin, since the beginning we have been able to express it using a simple function:

``` kotlin
val names = users.map { it.name }
```

Lazy property initialization pattern still cannot be expressed in Java. In Kotlin, we use a property delegate instead:

``` kotlin
val connection by lazy { makeConnection() }
```

Who knows how many other concepts are there, that we do not know how to extract and express directly. 

One of the features we should expect from a powerful programming language is the ability to build abstractions by assigning names to common patterns[2](chap65.xhtml#fn-SICP). In one of the most rudimentary forms, this is what we achieve by extracting functions, delegates, classes, etc. As a result, we can then work in terms of the abstractions directly. 

### Car metaphor

Many things happen when you drive a car. It requires the coordinated work of the engine, alternator, suspension and many other elements. Just imagine how hard driving a car would be if it required understanding and following each of these elements in real-time! Thankfully, it doesn’t. As a driver, all we need to know is how to use a car interface–the steering wheel, gear shifter, and pedals–to operate the vehicle. Everything under the hood can change. A mechanic can change from petrol to natural gas, and then to diesel, without us even knowing about it. As cars introduce more and more electronic elements and special systems, the interface remains the same for the most part. With such changes under the hood, the car’s performance would likely also change; however, we are able to operate the car regardless.

A car has a well-defined interface. Despite all of the complex components, it is simple to use. The steering wheel represents an abstraction for left-right direction change, the gear shifter is an abstraction for forward-backward direction change, the gas pedal an abstraction for acceleration, and the brake an abstraction of deceleration. These are all we need in an automobile. These are abstractions that hide all the magic happening under the hood. Thanks to that, users do not need to know anything about car construction. They only need to understand how to drive it. Similarly, creators or car enthusiasts can change everything in the car, and it is fine as long as driving stays the same. Remember this metaphor as we will refer to it throughout the chapter. 

Similarly, in programming, we use abstractions mainly to:

- Hide complexity
- Organize our code
- Give creators the freedom to change

The first reason was already described in *Chapter 3: Reusability* and I assume that it is clear at this point why it is important to extract functions, classes or delegates to reuse common logic or common algorithms. In *Item 26: Each function should be written in terms of a single level of abstraction*, we will see how to use abstractions to organize the code. In *Item 27: Use abstraction to protect code against changes*, we will see how to use abstractions to give ourselves the freedom to change. Then we will spend the rest of this chapter on creating and using abstractions. 

This is a pretty high-level chapter, and the rules presented here are a bit more abstract. Just after this chapter, we will cover some more concrete aspects of OOP design in *Chapter 5: Object creation* and *Chapter 6: Class design*. They will dive into deeper aspects of class implementation and use, but they will both build on this chapter.