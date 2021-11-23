When we define a state, we prefer to tighten the scope of variables and properties by:

- Using local variables instead of properties
- Using variables in the narrowest scope possible, so for instance, if a variable is used only inside a loop, defining it inside this loop

The scope of an element is the region of a computer program where the element is visible. In Kotlin, the scope is nearly always created by curly braces, and we can generally access elements from the outer scope. Take a look at this example:

```kotlin
val a = 1
fun fizz() {
   val b = 2
   print(a + b)
}
val buzz = {
   val c = 3
   print(a + c)
}
// Here we can see a, but not b nor c
```

In the above example, in the scope of the functions fizz and buzz, we can access variables from the outer scope. However, in the outer scope, we cannot access variables defined in those functions. Here is an example of how limiting variable scope might look like:

```kotlin
// Bad
var user: User
for (i in users.indices) {
   user = users[i]
   print("User at $i is $user")
}

// Better
for (i in users.indices) {
   val user = users[i]
   print("User at $i is $user")
}

// Same variables scope, nicer syntax
for ((i, user) in users.withIndex()) {
   print("User at $i is $user")
}
```

In the first example, the user variable is accessible not only in the scope of the for-loop, but also outside of it. In the second and third examples, we limit the scope of the user variable concretely to the scope of the for-loop.

Similarly, we might have many scopes inside scopes (most likely created by lambda expressions inside lambda expressions), and it is better to define variables in as narrow scope as possible.

There are many reasons why we prefer it this way, but the most important one is: When we tighten a variable’s scope, it is easier to keep our programs simple to track and manage. When we analyze code, we need to think about what elements are there at this point. The more elements there are to deal with, the harder it is to do programming. The simpler your application is, the less likely it will be to break. This is a similar reason to why we prefer immutable properties or objects over their mutable counterparts.

Thinking about mutable properties, it is easier to track how they change when they can only be modified in a smaller scope. It is easier to reason about them and change their behavior.

Another problem is that variables with a wider scope might be overused by another developer. For instance, one could reason that if a variable is used to assign the next elements in an iteration, the last element in the list should remain in that variable once the loop is complete. Such reasoning could lead to terrible abuse, like using this variable after the iteration to do something with the last element. It would be really bad because another developer trying to understand what value is there would need to understand the whole reasoning. This would be a needless complication.

Whether a variable is read-only or read-write, we always prefer a variable to be initialized when it is defined. Do not force a developer to look where it was defined. This can be supported with control structures such as if, when, try-catch or the Elvis operator used as expressions:

```kotlin
 // Bad
val user: User
if (hasValue) {
   user = getValue()
} else {
   user = User()
}

// Better
val user: User = if(hasValue) {
   getValue()
} else {
   User()
}
```

If we need to set-up multiple properties, destructuring declarations can help us:

```kotlin
// Bad
fun updateWeather(degrees: Int) {
   val description: String
   val color: Int
   if (degrees < 5) {
       description = "cold"
       color = Color.BLUE
   } else if (degrees < 23) {
       description = "mild"
       color = Color.YELLOW
   } else {
       description = "hot"
       color = Color.RED
   }
   // ...
}

// Better
fun updateWeather(degrees: Int) {
   val (description, color) = when {
       degrees < 5 -> "cold" to Color.BLUE
       degrees < 23 -> "mild" to Color.YELLOW
       else -> "hot" to Color.RED
   }
   // ...
}
```

Finally, too wide variable scope can be dangerous. Let’s describe one common danger.

#### Capturing

When I teach about Kotlin coroutines, one of my exercises is to implement the Sieve of Eratosthenes to find prime numbers using a sequence builder. The algorithm is conceptually simple:

1. We take a list of numbers starting from 2.

2. We take the first one. It is a prime number.

3. From the rest of the numbers, we remove the first one and we filter out all the numbers that are divisible by this prime number.

A very simple implementation of this algorithm looks like this:

```kotlin
var numbers = (2..100).toList()
val primes = mutableListOf<Int>()
while (numbers.isNotEmpty()) {
   val prime = numbers.first()
   primes.add(prime)
   numbers = numbers.filter { it % prime != 0 }
}
print(primes) // [2, 3, 5, 7, 11, 13, 17, 19, 23, 29, 31, 
// 37, 41, 43, 47, 53, 59, 61, 67, 71, 73, 79, 83, 89, 97]
```

The challenge is to let it produce a potentially infinite sequence of prime numbers. If you want to challenge yourself, stop now and try to implement it.

This is what the solution could look like:

```kotlin
val primes: Sequence<Int> = sequence {
   var numbers = generateSequence(2) { it + 1 }

   while (true) {
       val prime = numbers.first()
       yield(prime)
       numbers = numbers.drop(1)
           .filter { it % prime != 0 }
   }
}

print(primes.take(10).toList()) 
// [2, 3, 5, 7, 11, 13, 17, 19, 23, 29]
```

Although in nearly every group there is a person who tries to “optimize” it, and to not create the variable in every loop he or she extracts prime as a mutable variable:

```kotlin
val primes: Sequence<Int> = sequence {
   var numbers = generateSequence(2) { it + 1 }

   var prime: Int
   while (true) {
       prime = numbers.first()
       yield(prime)
       numbers = numbers.drop(1)
           .filter { it % prime != 0 }
   }
}
```

The problem is that this implementation does not work correctly anymore. These are the first 10 yielded numbers:

```kotlin
print(primes.take(10).toList()) 
// [2, 3, 5, 6, 7, 8, 9, 10, 11, 12]
```

Stop now and try to explain this result.

The reason why we have such a result is that we captured the variable prime. Filtering is done lazily because we’re using a sequence. In every step, we add more and more filters. In the “optimized” one we always add the filter which references the mutable property prime. Therefore we always filter the last value of prime. This is why this filtering does not work properly. Only dropping works and so we end up with consecutive numbers (except for 4 which was filtered out when prime was still set to 2).

We should be aware of problems with unintentional capturing because such situations happen. To prevent them we should avoid mutability and prefer narrower scope for variables.

#### Summary

For many reasons, we should prefer to define variables for the closest possible scope. Also, we prefer val over var also for local variables. We should always be aware of the fact that variables are captured in lambdas. These simple rules can save us a lot of trouble.

