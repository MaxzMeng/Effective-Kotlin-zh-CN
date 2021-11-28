## Item 43: Consider extracting non-essential parts of your API into extensions

When we define final methods in a class, we need to make a decision if we want to define them as members, or if we want to define them as extension functions. 

``` kotlin
// Defining methods as members
class Workshop(/*...*/) {
   //...

   fun makeEvent(date: DateTime): Event = //...

   val permalink
       get() = "/workshop/$name"
}
```

``` kotlin
// Defining methods as extensions
class Workshop(/*...*/) {
   //...
}

fun Workshop.makeEvent(date: DateTime): Event = //...

val Workshop.permalink
   get() = "/workshop/$name"
```

Both approaches are similar in many ways. Their use and even referencing them via reflection is very similar:

``` kotlin
fun useWorkshop(workshop: Workshop) {
   val event = workshop.makeEvent(date)
   val permalink = workshop.permalink
  
   val makeEventRef = Workshop::makeEvent
   val permalinkPropRef = Workshop::permalink
}
```

There are also significant differences though. They both have their pros and cons, and one way does not dominate over another. This is why my suggestion is to consider such extraction, not necessarily to do it. The point is to make smart decisions, and for that, we need to understand the differences. 

The biggest difference between members and extensions in terms of use is that **extensions need to be imported separately**. For this reason, they can be located in a different package. This fact is used when we cannot add a member ourselves. It is also used in projects designed to separate data and behavior. Properties with fields need to be located in a class, but methods can be located separately as long as they only access public API of the class. 

Thanks to the fact that extensions need to be imported **we can have many extensions with the same name on the same type**. This is good because different libraries can provide extra methods and we won’t have a conflict. On the other hand, it would be dangerous to have two extensions with the same name, but having different behavior. For such cases, we can cut the Gordian knot by making a member function. The compiler always chooses member functions over extensions[5](chap65.xhtml#fn-footnote_630_note).

Another significant difference is that **extensions are not virtual**, meaning that they cannot be redefined in derived classes. The extension function to call is selected statically during compilation. This is different behavior from member elements that are virtual in Kotlin. Therefore we should not use extensions for elements that are designed for inheritance. 

``` kotlin
open class C
class D: C()
fun C.foo() = "c"
fun D.foo() = "d"

fun main() {
   val d = D()
   print(d.foo()) // d
   val c: C = d
   print(c.foo()) // c

   print(D().foo()) // d
   print((D() as C).foo()) // c
}
```

This behavior is the result of the fact that extension functions under the hood are compiled into normal functions where the extension’s receiver is placed as the first argument: 

``` kotlin
fun foo(`this$receiver`: C) = "c"
fun foo(`this$receiver`: D) = "d"

fun main() {
   val d = D()
   print(foo(d)) // d
   val c: C =d
   print(foo(c)) // c

   print(foo(D())) // d
   print(foo(D() as C)) // c
}
```

Another consequence of this fact is that **we define extensions on types, not on classes**. This gives us more freedom. For instance, we can define an extension on a nullable or a concrete substitution of a generic type:

``` kotlin
inline fun CharSequence?.isNullOrBlank(): Boolean {
   contract {
       returns(false) implies (this@isNullOrBlank != null)
   }

   return this == null || this.isBlank()
}

public fun Iterable<Int>.sum(): Int {
   var sum: Int = 0
   for (element in this) {
       sum += element
   }
   return sum
}
```

The last important difference is that **extensions are not listed as members in the class reference**. This is why they are not considered by annotation processors and why, when we process a class using annotation processing, we cannot extract elements that should be processed into extension functions. On the other hand, if we extract non-essential elements into extensions, we don’t need to worry about them being seen by those processors. We don’t need to hide them, because they are not in the class anyway.

### Summary

The most important differences between members and extensions are:

- Extensions need to be imported
- Extensions are not virtual
- Member has a higher priority
- Extensions are on a type, not on a class
- Extensions are not listed in the class reference

To summarize it, extensions give us more freedom and flexibility. Although they do not support inheritance, annotation processing, and it might be confusing that they are not present in the class. Essential parts of our API should rather stay as members, but there are good reasons to extract non-essential parts of your API as extensions.