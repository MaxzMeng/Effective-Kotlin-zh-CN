## Item 25: Reuse between different platforms by extracting common modules

Companies rarely write applications only for just a single platform. They would rather develop a product for two or more platforms, and their products, nowadays, often rely on several applications running on different platforms. Think of client and server applications communicating through network calls. As they need to communicate, there are often similarities that can be reused. Implementations of the same product for different platforms generally have even more similarities. Especially their business logic is often nearly identical. These projects can profit significantly from sharing code.

### Full-stack development

Lots of companies are based on web development. Their product is a website, but in most cases, those products need a backend application (also called server-side). On websites, JavaScript is the king. It nearly has a monopoly on this platform. On the backend, a very popular option (if not the most popular) is Java. Since these languages are very different, it is common that backend and web development are separated. Things can change, however. Now Kotlin is becoming a popular alternative to Java for backend development. For instance with Spring, the most popular Java framework, Kotlin is a first-class citizen. Kotlin can be used as an alternative to Java in every framework. There are also many Kotlin backend frameworks like Ktor. This is why many backend projects migrate from Java to Kotlin. A great part of Kotlin is that it can also be compiled into JavaScript. There are already many Kotlin/JS libraries, and we can use Kotlin to write different kinds of web applications. For instance, we can write a web frontend using the React framework and Kotlin/JS. This allows us to write both the backend and the website all in Kotlin. What is even better, **we can have parts that compile both to JVM bytecode and to JavaScript.** Those are shared parts. We can place there, for instance, universal tools, API endpoint definitions, common abstractions, etc. that we might want to reuse. 

![img](../../assets/chapter3/chapter3-7.png)

### Mobile development

This capability is even more important in the mobile world. We rarely build only for Android. Sometimes we can live without a server, but we generally need to implement an iOS application as well. Each application is written for a different platform using different languages and tools. In the end, Android and iOS versions of the same application are very similar. They are often designed differently, but they nearly always have the same logic inside. Using Kotlin multiplatform capabilities, we can implement this logic only once and reuse it between those two platforms. We can make a common module and implement business logic there. Business logic should be independent of frameworks and platforms anyway (Clean Architecture). Such common logic can be written in pure Kotlin or using other common modules, and it can then be used on different platforms.

In Android, it can be used directly since Android is built the same way using Gradle. The experience is similar to having those common parts in our Android project.

For iOS, we compile these common parts to an Objective-C framework using Kotlin/Native which is compiled using LLVM. We can then use it from Swift in Xcode or AppCode. Alternatively, we can implement our whole application using Kotlin/Native. 

![](../../assets/chapter3/chapter3-8.png)

### Libraries

Defining common modules is also a powerful tool for libraries. In particular, those that do not depend strongly on platform can easily be moved to a common module and allow users to use them from all languages running on the JVM, JavaScript or natively (so from Java, Scala, JavaScript, CoffeeScript, TypeScript, C, Objective-C, Swift, Python, C#, etc.). 

### All together

We can use all those platforms together. Using Kotlin, we can develop for nearly all kinds of popular devices and platforms, and reuse code between them however we want. Just a few examples of what we can write in Kotlin:

- Backend in Kotlin/JVM, for instance on Spring or Ktor
- Website in Kotlin/JS, for instance in React
- Android in Kotlin/JVM, using the Android SDK
- iOS Frameworks that can be used from Objective-C or Swift, using Kotlin/Native
- Desktop applications in Kotlin/JVM, for instance in TornadoFX
- Raspberry Pi, Linux or Mac OS programs in Kotlin/Native 

Here is a typical application visualized:

![](../../assets/chapter3/chapter3-9.png)

We are still learning how to organize our code to make code reuse safe and efficient in the long run. But it is good to know the possibilities this approach gives us. We can reuse between different platforms using common modules. This is a powerful tool to eliminate redundancy and to reuse common logic and common algorithms.