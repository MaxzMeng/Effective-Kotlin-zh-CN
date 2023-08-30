## Item 38: Use function types instead of interfaces to pass operations and actions

Many languages do not have the concept of a function type. Instead, they use interfaces with a single method. Such interfaces are known as SAM’s (Single-Abstract Method). Here is an example SAM used to pass information about what should happen when a view is clicked:

``` kotlin
interface OnClick {
   fun clicked(view: View)
}
```

When a function expects a SAM, we must pass an instance of an object that implements this interface[1](chap65.xhtml#fn-footnote_610_note). 

``` kotlin
fun setOnClickListener(listener: OnClick) {
   //...
}

setOnClickListener(object : OnClick {
   override fun clicked(view: View) {
       // ...
   }
})
```

However, notice that declaring a parameter with a function type gives us much more freedom:

``` kotlin
fun setOnClickListener(listener: (View) -> Unit) {
   //... 
}
```

Now, we can pass the parameter as:

- A lambda expression or an anonymous function

``` kotlin
setOnClickListener { /*...*/ }
setOnClickListener(fun(view) { /*...*/ })
```

- A function reference or bounded function reference

``` kotlin
setOnClickListener(::println)
setOnClickListener(this::showUsers)
```

- Objects that implement the declared function type

``` kotlin
class ClickListener: (View)->Unit {
   override fun invoke(view: View) {
       // ...
   }
}

setOnClickListener(ClickListener())
```

These options can cover a wider spectrum of use cases. On the other hand, one might argue that the advantage of a SAM is that it and its arguments are named. Notice that we can name function types using type aliases as well. 

``` kotlin
typealias OnClick = (View) -> Unit
```

Parameters can also be named. The advantage of naming them is that these names can then be suggested by default by an IDE.

``` kotlin
fun setOnClickListener(listener: OnClick) { /*...*/ }
typealias OnClick = (view: View)->Unit
```

![](../../assets/chapter6/chapter6-3.png)

Notice that when we use lambda expressions, we can also destructure arguments. Together, this makes function types generally a better option than SAMs. 

This argument is especially true when we set many observers. The classic Java way often is to collect them in a single listener interface:

``` kotlin
class CalendarView {
   var listener: Listener? = null

   interface Listener {
       fun onDateClicked(date: Date)
       fun onPageChanged(date: Date)
   }
}
```

I believe this is largely a result of laziness. From an API consumer’s point of view, it is better to set them as separate properties holding function types:

``` kotlin
class CalendarView {
   var onDateClicked: ((date: Date) -> Unit)? = null
   var onPageChanged: ((date: Date) -> Unit)? = null
}
```

This way, the implementations of `onDateClicked` and `onPageChanged` do not need to be tied together in an interface. Now, these functions may be changed independently. 

If you don’t have a good reason to define an interface, prefer using function types. They are well supported and are used frequently by Kotlin developers. 

### When should we prefer a SAM?

There is one case when we prefer a SAM: When we design a class to be used from another language than Kotlin. Interfaces are cleaner for Java clients. They cannot see type aliases nor name suggestions. Finally, Kotlin function types when used from some languages (especially Java) require functions to return `Unit` explicitly:

``` kotlin
// Kotlin
class CalendarView() {
   var onDateClicked: ((date: Date) -> Unit)? = null
   var onPageChanged: OnDateClicked? = null
}

interface OnDateClicked {
   fun onClick(date: Date)
}

// Java
CalendarView c = new CalendarView();
c.setOnDateClicked(date -> Unit.INSTANCE);
c.setOnPageChanged(date -> {});
```

This is why it might be reasonable to use SAM instead of function types when we design API for use from Java. Though in other cases, prefer function types.