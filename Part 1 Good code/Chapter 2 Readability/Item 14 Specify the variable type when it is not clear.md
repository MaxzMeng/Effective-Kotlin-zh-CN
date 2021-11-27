## Item 14: Specify the variable type when it is not clear

Kotlin has a great type inference system that allows us to omit types when those are obvious for developers:

``` kotlin
val num = 10
val name = "Marcin"
val ids = listOf(12, 112, 554, 997)
```

It improves not only development time, but also readability when a type is clear from the context and additional specification is redundant and messy. However, it should not be overused in cases when a type is not clear:

``` kotlin
val data = getSomeData()
```

We design our code for readability, and we should not hide important information that might be important for a reader. It is not valid to argue that return types can be omitted because the reader can always jump into the function specification to check it there. Type might be inferred there as well and a user can end up going deeper and deeper. Also, a user might read this code on GitHub or some other environment that does not support jumping into implementation. Even if they can, we all have very limited working memory and wasting it like this is not a good idea. Type is important information and if it is not clear, it should be specified. 

``` kotlin
val data: UserData = getSomeData()
```

Improving readability is not the only case for type specification. It is also for safety as shown in the *Chapter: Safety* on *Item 3: Eliminate platform types as soon as possible* and *Item 4: Do not expose inferred types*. **Type might be important information both for a developer and for the compiler. Whenever it is, do not hesitate to specify the type. It costs little and can help a lot.**