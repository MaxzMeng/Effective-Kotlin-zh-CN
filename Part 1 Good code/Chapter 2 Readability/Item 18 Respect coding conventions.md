## Item 18: Respect coding conventions

Kotlin has well-established coding conventions described in the documentation in a section aptly called “Coding Conventions”. **Those conventions are not optimal for all projects, but it is optimal for us as a community to have conventions that are respected in all projects.** Thanks to them:

- It is easier to switch between projects
- Code is more readable even for external developers
- It is easier to guess how code works
- It is easier to later merge code with a common repository or to move some parts of code from one project to another

Programmers should get familiar with those conventions as they are described in the documentation. They should also be respected when they change - which might happen to some degree over time. Since it is hard to do both, there are two tools that help:

- The IntelliJ formatter can be set up to automatically format according to the official Coding Conventions style. For that go to Settings | Editor | Code Style | Kotlin, click on “Set from…” link in the upper right corner, and select “Predefined style / Kotlin style guide” from the menu.
- ktlint - popular linter that analyzes your code and notifies you about all coding conventions violations.

Looking at Kotlin projects, I see that most of them are intuitively consistent with most of the conventions. This is probably because Kotlin mostly follows the Java coding conventions, and most Kotlin developers today are post-Java developers. One rule that I see often violated is how classes and functions should be formatted. According to the conventions, classes with a short primary-constructor can be defined in a single line:

```
class FullName(val name: String, val surname: String)
```

However, classes with many parameters should be formatted in a way so that every parameter is on another line, and there is no parameter in the first line:

``` kotlin
class Person(
    val id: Int = 0,
    val name: String = "",
    val surname: String = ""
) : Human(id, name) { 
    // body
}
```

Similarly, this is how we format a long function:

``` kotlin
public fun <T> Iterable<T>.joinToString(
    separator: CharSequence = ", ", 
    prefix: CharSequence = "", 
    postfix: CharSequence = "", 
    limit: Int = -1, 
    truncated: CharSequence = "...", 
    transform: ((T) -> CharSequence)? = null
): String {
   // ...
}
```

Notice that those two are very different from the convention that leaves the first parameter in the same line and then indents all others to it. 

``` kotlin
// Don’t do that
class Person(val id: Int = 0,
             val name: String = "",
             val surname: String = "") : Human(id, name){ 
    // body
}
```

It can be problematic in 2 ways:

- Arguments on every class start with a different indentation based on the class name. Also, when we change the class name, we need to adjust the indentations of all primary constructor parameters.
- Classes defined this way tend to be still too wide. Width of the class defined this way is the class name with `class` keyword and the longest primary constructor parameter, or last parameter plus superclasses and interfaces. 

Some teams might decide to use slightly different conventions. This is fine, but then those conventions should be respected all around the given project. **Every project should look like it was written by a single person, not a group of people fighting with each other.**

Coding conventions are often not respected enough by developers, but they are important, and a chapter dedicated to readability in a best practices book couldn’t be closed without at least a short section dedicated to them. Read them, use static checkers to help you be consistent with them, apply them in your projects. By respecting coding conventions, we make Kotlin projects better for us all.