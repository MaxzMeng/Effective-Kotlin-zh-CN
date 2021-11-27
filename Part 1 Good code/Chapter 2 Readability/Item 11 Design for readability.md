## Item 11: Design for readability

It is a known observation in programming that developers read code much more than they write it. A common estimate is that for every minute spent writing code, ten minutes are spent reading it (this ratio was popularized by Robert C. Martin in the book *Clean Code*). If this seems unbelievable, just think about how much time you spend on reading code when you are trying to find an error. I believe that everyone has been in this situation at least once in their career where they’ve been searching for an error for weeks, just to fix it by changing a single line. When we learn how to use a new API, it’s often from reading code. We usually read the code to understand what is the logic or how implementation works. **Programming is mostly about reading, not writing.** Knowing that it should be clear that we should code with readability in mind.

### Reducing cognitive load

Readability means something different to everyone. However, there are some rules that were formed based on experience or came from cognitive science. Just compare the following two implementations:

``` kotlin
// Implementation A
if (person != null && person.isAdult) {
   view.showPerson(person)
} else {
   view.showError()
}

// Implementation B
person?.takeIf { it.isAdult }
   ?.let(view::showPerson)
   ?: view.showError()
```

Which one is better, A or B? Using the naive reasoning, that the one with fewer lines is better, is not a good answer. We could just as well remove the line breaks from the first implementation, and it wouldn’t make it more readable. 

How readable both constructs are, depends on how fast we can understand each of them. This, in turn, depends a lot on how much our brain is trained to understand each idiom (structure, function, pattern). For a beginner in Kotlin, surely implementation A is way more readable. It uses general idioms (if/else, `&&`, method calls). Implementation B has idioms that are typical to Kotlin (safe call `?.`, `takeIf`, `let`, Elvis operator `?:`, a bounded function reference `view::showPerson`). Surely, all those idioms are commonly used throughout Kotlin, so they are well known by most experienced Kotlin developers. Still, it is hard to compare them. Kotlin isn’t the first language for most developers, and we have much more experience in general programming than in Kotlin programming. We don’t write code only for experienced developers. The chances are that the junior you hired (after fruitless months of searching for a senior) does not know what `let`, `takeIf`, and bounded references are. It is also very likely that they never saw the Elvis operator used this way. That person might spend a whole day puzzling over this single block of code. Additionally, even for experienced Kotlin developers, Kotlin is not the only programming language they use. Many developers reading your code will not be experienced with Kotlin. The brain will always need to spend a bit of time to recognize Kotlin-specific idioms. Even after years with Kotlin, it still takes much less time for me to understand the first implementation. Every less known idiom introduces a bit of complexity and when we analyze them all together in a single statement that we need to comprehend nearly all at once, this complexity grows quickly.

Notice that implementation A is easier to modify. Let’s say that we need to add additional operation on the `if` block. In the implementation A adding that is easy. In the implementation B we cannot use function reference anymore. Adding something to the `else` block in the implementation B is even harder - we need to use some function to be able to hold more than a single expression on the right side of the Elvis operator:

``` kotlin
if (person != null && person.isAdult) {
   view.showPerson(person)
   view.hideProgressWithSuccess()
} else {
   view.showError()
   view.hideProgress()
}

person?.takeIf { it.isAdult }
   ?.let {
       view.showPerson(it)
       view.hideProgressWithSuccess()
   } ?: run {
       view.showError()
       view.hideProgress()
   }
```

Debugging implementation A is also much simpler. No wonder why - debugging tools were made for such basic structures. 

The general rule is that less common and “creative” structures are generally less flexible and not so well supported. Let’s say for instance that we need to add a third branch to show different error when person is `null` and different one when he or she is not an adult. On the implementation A using `if`/`else`, we can easily change `if`/`else` to `when` using IntelliJ refactorization, and then easily add additional branch. The same change on the code would be painful on the implementation B. It would probably need to be totally rewritten.

Have you noticed that implementation A and B do not even work the same way? Can you spot the difference? Go back and think about it now. 

The difference lies in the fact that `let` returns a result from the lambda expression. This means that if `showPerson` would return `null`, then the second implementation would call `showError` as well! This is definitely not obvious, and it teaches us that when we use less familiar structures, it is easier to fall in unexpected behavior (and not to spot them).

The general rule here is that we want to reduce cognitive load. Our brains recognize patterns and based on these patterns they build our understanding of how programs work. When we think about readability we want to shorten this distance. We prefer less code, but also more common structures. We recognize familiar patterns when we see them often enough. We always prefer structures that we are familiar with in other disciplines.

### Do not get extreme

Just because in the previous example I presented how `let` can be misused, it does not mean that it should be always avoided. It is a popular idiom reasonably used in a variety of contexts to actually make code better. One popular example is when we have a nullable mutable property and we need to do an operation only if it is not null. Smart casting cannot be used because mutable property can be modified by another thread. One great way to deal with that is to use safe call `let`:

``` kotlin
class Person(val name: String)
var person: Person? = null

fun printName() {
    person?.let {
        print(it.name)
    }
}
```

Such idiom is popular and widely recognizable. There are many more reasonable cases for `let`. For instance:

- To move operation after its argument calculation
- To use it to wrap an object with a decorator

Here are examples showing those two (both additionally use function references):

``` kotlin
students
     .filter { it.result >= 50 }	
     .joinToString(separator = "\n") { 
        "${it.name} ${it.surname}, ${it.result}" 
     }
     .let(::print)

var obj = FileInputStream("/file.gz")
    .let(::BufferedInputStream)
    .let(::ZipInputStream)
    .let(::ObjectInputStream)
    .readObject() as SomeObject
```

In all those cases we pay our price - this code is harder to debug and harder to be understood by less experienced Kotlin developers. But we pay for something and it seems like a fair deal. The problem is when we introduce a lot of complexity for no good reason. 

There will always be discussions when something makes sense and when it does not. Balancing that is an art. It is good though to recognize how different structures introduce complexity or how they clarify things. Especially since when they are used together, the complexity of two structures is generally much more than the sum of their individual complexities. 

### Conventions

We’ve acknowledged that different people have different views of what readability means. We constantly fight over function names, discuss what should be explicit and what implicit, what idioms should we use, and much more. Programming is an art of expressiveness. Still, there are some conventions that need to be understood and remembered. 

When one of my workshop groups in San Francisco asked me about the worst thing one can do in Kotlin, I gave them this:

``` kotlin
val abc = "A" { "B" } and "C"
print(abc) // ABC
```

All we need to make this terrible syntax possible is the following code:

``` kotlin
operator fun String.invoke(f: ()->String): String = 
    this + f()

infix fun String.and(s: String) = this + s
```

This code violates many rules that we will describe later:

- It violates operator meaning - `invoke` should not be used this way. A `String` cannot be invoked.
- The usage of the ‘lambda as the last argument’ convention here is confusing. It is fine to use it after functions, but we should be very careful when we use it for the `invoke` operator. 
- `and` is clearly a bad name for this infix method. `append` or `plus` would be much better. 
- We already have language features for `String` concatenation and we should use them instead of reinventing the wheel. 

Behind each of these suggestions, there is a more general rule that guards good Kotlin style. We will cover the most important ones in this chapter starting with the first item which will focus on overriding operators.