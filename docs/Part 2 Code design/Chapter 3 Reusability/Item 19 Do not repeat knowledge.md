## Item 19: Do not repeat knowledge

The first big rule I was taught about programming was:

If you use copy-paste in your project, you are most likely doing something wrong.

This is a very simple heuristic, but it is also very wise. Till today whenever I reflect on that, I am amazed how well a single and clear sentence expresses the key idea behind the “Do not repeat knowledge” principle. It is also often known as DRY principle after the Pragmatic Programmer book that described the Don’t Repeat Yourself rule. Some developers might be familiar with the WET antipattern, that sarcastically teaches us the same. DRY is also connected to the Single Source of Truth (SSOT) practice, As you can see, this rule is quite popular and has many names. However, it is often misused or abused. To understand this rule and the reasons behind it clearly, we need to introduce a bit of theory.

### Knowledge

Let’s define knowledge in programming broadly, as any piece of intentional information. It can be stated by code or data. It can also be stated by lack of code or data, which means that we want to use the default behavior. For instance when we inherit, and we don’t override a method, it’s like saying that we want this method to behave the same as in the superclass. 

With knowledge defined this way, everything in our projects is some kind of knowledge. Of course, there are many different kinds of knowledge: how an algorithm should work, what UI should look like, what result we wish to achieve, etc. There are also many ways to express it: for example by using code, configurations, or templates. In the end, every single piece of our program is information that can be understood by some tool, virtual machine, or directly by other programs.

There are two particularly important kinds of knowledge in our programs:

1. Logic - How we expect our program to behave and what it should look like
2. Common algorithms - Implementation of algorithms to achieve the expected behavior

The main difference between them is that business logic changes a lot over time, while common algorithms generally do not once they are defined. They might be optimized or we might replace one algorithm with another, but algorithms themselves are generally stable. Because of this difference, we will concentrate on algorithms in the next item. For now, let’s concentrate on the first point, that is the logic - knowledge about our program. 

### Everything can change

There is a saying that in programming the only constant is change. Just think of projects from 10 or 20 years ago. It is not such a long time. Can you point a single popular application or website that hasn’t changed? Android was released in 2008. The first stable version of Kotlin was released in 2016. Not only technologies but also languages change so quickly. Think about your old projects. Most likely now you would use different libraries, architecture, and design. 

Changes often occur where we don’t expect them. There is a story that once when Einstein was examining his students, one of them stood up and loudly complained that questions were the same as the previous year. Einstein responded that it was true, but answers were totally different that year. Even things that you think are constant, because they are based on law or science, might change one day. Nothing is absolutely safe.

Standards of UI design and technologies change much faster. Our understanding of clients often needs to change on a daily basis. This is why knowledge in our projects will change as well. For instance, here are very typical reasons for the change:

- The company learns more about user needs or habits
- Design standards change
- We need to adjust to changes in the platform, libraries, or some tools

Most projects nowadays change requirements and parts of the internal structure every few months. This is often something desired. Many popular management systems are agile and fit to support constant changes in requirements. Slack was initially a game named Glitch[3](chap65.xhtml#fn-footnote_32_note). The game didn’t work out, but customers liked its communication features. 

Things change, and we should be prepared for that. **The biggest enemy of changes is knowledge repetition.** Just think for a second: what if we need to change something that is repeated in many places in our program? The simplest answer is that in such a case, you just need to search for all the places where this knowledge is repeated, and change it everywhere. Searching can be frustrating, and it is also troublesome: what if you forget to change some repetitions? What if some of them are already modified because they were integrated with other functionalities? It might be tough to change them all in the same way. Those are real problems. 

To make it less abstract, think of a universal button used in many different places in our project. When our graphic designer decides that this button needs to be changed, we would have a problem if we defined how it looks like in every single usage. We would need to search our whole project and change every single instance separately. We would also need to ask testers to check if we haven’t missed any instance. 

Another example: Let’s say that we use a database in our project, and then one day we change the name of a table. If we forget to adjust all SQL statements that depend on this table, we might have a very problematic error. If we had some table structure defined only once, we wouldn’t have such a problem. 

On both examples, you can see how dangerous and problematic knowledge repetition is. It makes projects less scalable and more fragile. Good news is that we, programmers, work for years on tools and features that help us eliminate knowledge redundancy. On most platforms, we can define a custom style for a button, or custom view/component to represent it. Instead of writing SQL in text format, we can use an ORM (like Hibernate) or DAO (like Exposed).

All those solutions represent different kinds of abstractions and they protect us from a different kinds of redundancy. Analysis of different kinds of abstractions is presented in *Item 27: Use abstraction to protect code against changes*. 

### When should we allow code repetition?

There are situations where we can see two pieces of code that are similar but should not be extracted into one. This is when they only look similar but represent different knowledge. 

Let’s start from an example. Let’s say that we have two independent Android applications in the same project. Their build tool configurations are similar so it might be tempting to extract it. But what if we do that? Applications are independent so if we will need to change something in the configuration, we will most likely need to change it only in one of them. Changes after this reckless extraction are harder, not easier. Configuration reading is harder as well - configurations have their boilerplate code, but developers are already familiar with it. Making abstractions means designing our own API, which is another thing to learn for a developer using this API. This is a perfect example of how problematic it is when we extract something that is not conceptually the same knowledge.

The most important question to ask ourselves when we decide if two pieces of code represent similar knowledge is: **Are they more likely going to change together or separately?** Pragmatically this is the most important question because this is the biggest result of having a common part extracted: it is easier to change them both, but it is harder to change only a single one. 

One useful heuristic is that if business rules come from different sources, we should assume that they will more likely change independently. For such a case we even have a rule that protects us from unintended code extraction. It is called the *Single Responsibility Principle*. 

### Single responsibility principle

A very important rule that teaches us when we should not extract common code is the Single Responsibility Principle from SOLID. It states that “A class should have only one reason to change”. This rule[4](chap65.xhtml#fn-footnote_33_note) can be simplified by the statement that there should be no such situations when two actors need to change the same class. By actor, we mean a source of change. They are often personified by developers from different departments who know little about each other’s work and domain. Although even if there is only a single developer in a project, but having multiple managers, they should be treated as separate actors. Those are two sources of changes knowing little about each other domains. The situation when two actors edit the same piece of code is especially dangerous.

Let’s see an example. Imagine that we work for a university, and we have a class `Student`. This class is used both by the Scholarships Department and the Accreditations Department. Developers from those two departments introduced two different properties:

- `isPassing` was created by the Accreditations Department and answers the question of whether a student is passing.
- `qualifiesForScholarship` was created by the Scholarships Department and answers the question if a student has enough points to qualify for a Scholarship.

Both functions need to calculate how many points the student collected in the previous semester, so a developer extracted a function `calculatePointsFromPassedCourses`.

![](../../assets/chapter3/chapter3-1.png)

``` kotlin
class Student {
   // ...
  
   fun isPassing(): Boolean = 
       calculatePointsFromPassedCourses() > 15
  
   fun qualifiesForScholarship(): Boolean = 
       calculatePointsFromPassedCourses() > 30
  
   private fun calculatePointsFromPassedCourses(): Int {
       //...
   }
}
```

Then, original rules change and the dean decides that less important courses should not qualify for scholarship points calculation. A developer who was sent to introduce this change checked function `qualifiesForScholarship`, finds out that it calls the private method `calculatePointsFromPassedCourses` and changes it to skip courses that do not qualify. Unintentionally, that developer changed the behavior of `isPassing` as well. Students who were supposed to pass, got informed that they failed the semester. You can imagine their reaction.

It is true that we could easily prevent such situation if we would have unit tests (Item 10: Write unit tests), but let’s skip this aspect for now. 

The developer might check where else the function is used. Although the problem is that this developer didn’t expect that this private function was used by another property with a totally different responsibility. Private functions are rarely used just by more than one function. 

This problem, in general, is that it is easy to couple responsibilities located very close (in the same class/file). A simple solution would be to extract these responsibilities into separate classes. We might have separate classes `StudentIsPassingValidator` and 
`StudentQualifiesForScholarshipValidator`. Though in Kotlin we don’t need to use such heavy artillery (see more at *Chapter 4: Design abstractions*). We can just define `qualifiesForScholarship` and 
`calculatePointsFromPassedCourses` as extension functions on `Student` located in separate modules: one over which Scholarships Department is responsible, and another over which Accreditations Department is responsible.

``` kotlin
// scholarship module
fun Student.qualifiesForScholarship(): Boolean { 
    /*...*/ 
}

// accreditations module
fun Student.calculatePointsFromPassedCourses(): Boolean { 
    /*...*/ 
}
```

What about extracting a function for calculating results? We can do it, but it cannot be a private function used as a helper for both these methods. Instead, it can be:

1. A general public function defined in a module used by both departments. In such a case, the common part is treated as something common, so a developer should not change it without modifying the contract and adjusting usages.
2. Two separate helper functions, each for every department.

Both options are safe.

The *Single Responsibility Principle* teaches us two things:

1. Knowledge coming from two different sources (here two different departments) is very likely to change independently, and we should rather treat it as a different knowledge.
2. We should separate different knowledge because otherwise, it is tempting to reuse parts that should not be reused. 

### Summary

Everything changes and it is our job to prepare for that: to recognize common knowledge and extract it. If a bunch of elements has similar parts and it is likely that we will need to change it for all instances, extract it and save time on searching through the project and update many instances. On the other hand, protect yourself from unintentional modifications by separating parts that are coming from different sources. Often it’s even more important side of the problem. I see many developers who are so terrified of the literal meaning of Don’t Repeat Yourself, that they tend to looking suspiciously at any 2 lines of code that look similar. Both extremes are unhealthy, and we need to always search for a balance. Sometimes, it is a tough decision if something should be extracted or not. This is why it is an art to design information systems well. It requires time and a lot of practice.