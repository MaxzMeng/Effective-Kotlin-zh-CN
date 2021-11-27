## Item 10: Write unit tests

In this chapter, you’ve seen quite a few ways to make your code safer, but the ultimate way to achieve this is to have different kinds of tests. One kind is checking that our application behaves correctly from the user’s perspective. These kinds of tests are too often the only ones recognized by management as this is generally their primary goal to make the application behave correctly from outside, not internally. These kinds of tests do not even need developers at all. They can be handled by a sufficient number of testers or, what is generally better in the long run, by automatic tests written by test engineers.

Such tests are useful for programmers, but they are not sufficient. They do not build proper assurance that concrete elements of our system behave correctly. They also do not provide fast feedback that is useful during development. For that, we need a different kind of tests that is much more useful for developers, and that is written by developers: unit tests. Here is an example unit test checking if our function `fib` calculating the Fibonacci number at n-th position gives us correct results for the first 5 numbers:

``` kotlin
@Test
fun `fib works correctly for the first 5 positions`() {
   assertEquals(1, fib(0))
   assertEquals(1, fib(1))
   assertEquals(2, fib(2))
   assertEquals(3, fib(3))
   assertEquals(5, fib(4))
}
```

With unit tests, we typically check:

- Common use cases (the happy path) - typical ways we expect the element to be used. Just like in the example above, we test if the function works for a few small numbers.
- Common error cases or potential problems - Cases that we suppose might not work correctly or that were shown to be problematic in the past. 
- Edge-cases and illegal arguments - for `Int` we might check for really big numbers like `Int.MAX_VALUE`. For a nullable object, it might be `null` or object filled with `null` values. There are no Fibonacci numbers for negative positions, so we might check how this function behaves then. 

Unit tests can be really useful during development as they give fast feedback on how the implemented element works. Tests are only ever-accumulating so you can easily check for regression. They can also check cases that are hard to test manually. There is even an approach called Test Driven Development (TDD) in which we write a unit test first and then implementation to satisfy it[10](chap65.xhtml#fn-unittests). 

The biggest advantages that result from unit tests are:

- **Well-tested elements tend to be more reliable.** There is also a psychological safety. When elements are well tested, we operate more confidently on them. 
- **When an element is properly tested, we are not afraid to refactor it.** As a result, well-tested programs tend to get better and better. On the other hand, in programs that are not tested, developers are scared of touching legacy code because they might accidentally introduce an error without even knowing about it. 
- **It is often much faster to check if something works correctly using unit tests rather than checking it manually.** A faster feedback-loop makes development faster and more pleasurable[11](chap65.xhtml#fn-footnote_141_note). It also helps reduce the cost of fixing bugs: the quicker you find them, the cheaper it is to fix them.

Clearly, there are also disadvantages to unit tests:

- It takes time to write unit tests. Though **in the long-term, good unit tests rather save our time as we spend less time debugging and looking for bugs later.** We also save a lot of time as running unit tests is much faster than manual testing or other kinds of automated tests. 
- We need to adjust our code to make it testable. Such changes are often hard, but they generally also force developers to use good and well-established architectures. 
- It is hard to write good unit tests. It requires skills and understanding that are orthogonal to the rest of the development. Poorly written unit tests can do more harm than good. **Everyone needs to learn how to properly unit-test their code.** It is useful to take a course on Software-Testing or Test Driven Development (TDD) first.

The biggest challenge is to obtain the skills to effectively unit test and to write code that supports unit testing. **Experienced Kotlin developers should obtain such skills and learn to unit test at least the important parts of the code.** Those are:

- Complex functionalities
- Parts that will most likely change over time or will be refactored
- Business logic
- Parts of our public API
- Parts that have a tendency to break
- Production bugs that we fixed

We do not need to stop there. Tests are an investment in application reliability and long-term maintainability.

### Summary

This chapter was started with a reflection that the first priority should be for our programs to behave correctly. It can be supported by using good practices presented in this chapter, but above that, the best way to ensure that our application behaves correctly is to check it by testing, especially unit testing. This is why a responsible chapter about safety needs at least a short section about unit testing. Just like responsible business application requires at least some unit tests.