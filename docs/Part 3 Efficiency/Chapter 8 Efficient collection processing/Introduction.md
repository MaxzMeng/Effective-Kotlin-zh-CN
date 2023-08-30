# Chapter 8: Efficient collection processing

Collections are one of the most important concepts in programming. In iOS, one of the most important view elements, `UICollectionView`, is designed to represent a collection. Similarly, in Android, it is hard to imagine an application without `RecyclerView` or `ListView`. When you need to write a portal with news, you will have a list of news. Each of them will probably have a list of authors and a list of tags. When you make an online shop, you start from a list of products. They will most likely have a list of categories and a list of different variants. When a user buys, they use some basket which is probably a collection of products and amounts. Then they need to choose from a list of delivery options and a list of payment methods. In some languages, `String` is just a list of characters. Collections are everywhere in programming! Just think about your application and you will quickly see lots of collections. 

This fact can be reflected also in programming languages. Most modern languages have some collection literals:

``` python
1 // Python
2 primes = [2, 3, 5, 7, 13]
3 // Swift
4 let primes = [2, 3, 5, 7, 13]
```

Good collection processing was a flag functionality of functional programming languages. Name of the Lisp programming language[1](chap65.xhtml#fn-footnote_80_note) stands for “list processing”. Most modern languages have good support for collection processing. This statement includes Kotlin which has one of the most powerful sets of tools for collection processing. Just think of the following collection processing:

``` kotlin
val visibleNews = mutableListOf<News>()
for (n in news) {
   if(n.visible) {
       visibleNews.add(n)
   }
}

Collections.sort(visibleNews, 
    { n1, n2 -> n2.publishedAt - n1.publishedAt })
val newsItemAdapters = mutableListOf<NewsItemAdapter>()
for (n in visibleNews) {
   newsItemAdapters.add(NewsItemAdapter(n))
}
```

In Kotlin it can be replaced with the following notation:

``` kotlin
val newsItemAdapters = news
       .filter { it.visible }
       .sortedByDescending { it.publishedAt }
       .map(::NewsItemAdapter)
```

Such notation is not only shorter, but it is also more readable. Every step makes a concrete transformation on the list of elements. Here is a visualization of the above processing:

![](../../assets/chapter8/chapter8-1.png)

The performance of the above examples is very similar. It is not always so simple though. Kotlin has a lot of collection processing methods and we can do the same processing in a lot of different ways. For instance, the below processing implementations have the same result but different performance:

``` kotlin
fun productsListProcessing(): String {
   return clientsList
           .filter { it.adult }
           .flatMap { it.products }
           .filter { it.bought }
           .map { it.price }
           .filterNotNull()
           .map { "$$it" }
           .joinToString(separator = " + ")
}

fun productsSequenceProcessing(): String {
   return clientsList.asSequence()
           .filter { it.adult }
           .flatMap { it.products.asSequence() }
           .filter { it.bought }
           .mapNotNull { it.price }
           .joinToString(separator = " + ") { "$$it" }
}
```

Collection processing optimization is much more than just a brain-puzzle. Collection processing is extremely important and, in big systems, often performance-critical. This is why it is often key to improve the performance of our program. Especially if we do backend application development or data analysis. Although, when we are implementing front-end clients, we can face collection processing that limits the performance of our application as well. As a consultant, I’ve seen a lot of projects and my experience is that I see collection processing again and again in lots of different places. This is not something that can be ignored so easily. 

The good news is that collection processing optimization is not hard to master. There are some rules and a few things to remember but actually, anyone can do it effectively. This is what we are going to learn in this chapter.