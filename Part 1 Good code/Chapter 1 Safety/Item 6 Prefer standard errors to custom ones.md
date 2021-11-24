## Item 6: Prefer standard errors to custom ones

Functions `require`, `check` and `assert` cover the most common Kotlin errors, but there are also other kinds of unexpected situations we need to indicate. For instance, when you implement a library to parse the JSON format, it is reasonable to throw a `JsonParsingException` when the provided JSON file does not have the correct format:

``` kotlin
inline fun <reified T> String.readObject(): T {
   //...
   if (incorrectSign) {
       throw JsonParsingException()
   }
   //...
   return result
}
```

Here we used a custom error because there is no suitable error in the standard library to indicate that situation. Whenever possible, you should use exceptions from the standard library instead of defining your own. Such exceptions are known by developers and they should be reused. Reusing known elements with well-established contracts makes your API easier to learn and to understand. Here is the list of some of the most common exceptions you can use:

- `IllegalArgumentException` and `IllegalStateException` that we throw using `require` and `check` as described in Item 5: Specify your expectations on arguments and state.
- `IndexOutOfBoundsException` - Indicate that the index parameter value is out of range. Used especially by collections and arrays. It is thrown for instance by `ArrayList.get(Int)`.
- `ConcurrentModificationException` - Indicate that concurrent modification is prohibited and yet it has been detected. 
- `UnsupportedOperationException` - Indicate that the declared method is not supported by the object. Such a situation should be avoided and when a method is not supported, it should not be present in the class. 
- `NoSuchElementException` - Indicate that the element being requested does not exist. Used for instance when we implement `Iterable` and the client asks for `next` when there are no more elements.