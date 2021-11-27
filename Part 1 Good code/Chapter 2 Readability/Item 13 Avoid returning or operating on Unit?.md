## Item 13: Avoid returning or operating on `Unit?`

During the recruitment process, a dear friend of mine was asked, “Why might one want to return `Unit?` from a function?”. Well, `Unit?` has only 2 possible values: `Unit` or `null`. Just like `Boolean` can be either `true` or `false`. Ergo, these types are isomorphic, so they can be used interchangeably. Why would we want to use `Unit?` instead of `Boolean` to represent something? I have no other answer than that one can use the Elvis operator or a safe call. So instead of:

``` kotlin
fun keyIsCorrect(key: String): Boolean = //...

if(!keyIsCorrect(key)) return
```

We are able to do this:

``` kotlin
fun verifyKey(key: String): Unit? = //...

verifyKey(key) ?: return
```

This appears to be the expected answer. What was missing in my friend’s interview was a way more important question: “But should we do it?”. This trick looks nice when we are writing the code, but not necessarily when we are reading it. Using `Unit?` to represent logical values is misleading and can lead to errors that are hard to detect. We’ve already discussed how this expression can be surprising:

``` kotlin
getData()?.let{ view.showData(it) } ?: view.showError()
```

When `showData` returns `null` and `getData` returns not `null`, both `showData` and `showError` will be called. Using standard if-else is less tricky and more readable:

``` kotlin
if (person != null && person.isAdult) {
   view.showPerson(person)
} else {
   view.showError()
}
```

Compare the following two notations:

``` kotlin
if(!keyIsCorrect(key)) return

verifyKey(key) ?: return
```

I have never found even a single case when `Unit?` is the most readable option. It is misleading and confusing. It should nearly always be replaced by `Boolean`. This is why the general rule is that we should avoid returning or operating on `Unit?`.