# Chapter 7: Make it cheap

Code efficiency today is often treated indulgently. To a certain degree, it is reasonable. Memory is cheap and developers are expensive. Though if your application is running on millions of devices, it consumes a lot of energy, and some optimization of battery use might save enough energy to power a small city. Or maybe your company is paying lots of money for servers and their maintenance, and some optimization might make it significantly cheaper. Or maybe your application works well for a small number of requests but does not scale well and on the day of the trial, it shuts down. Customers remember such situations. 

Efficiency is important in the long term, but optimization is not easy. Premature optimization often does more harm than good. Instead, there are some rules that can help you make more efficient programs nearly painlessly. Those are the cheap wins: they cost nearly nothing, but still, they can help us improve performance significantly. When they are not sufficient, we should use a profiler and optimize performance-critical parts. This is more difficult to achieve because it requires more understanding of what is expensive and how some optimizations can be done. 

This and the next chapter are about performance:

- *Chapter 7: Make it cheap* - more general suggestions for performance. 
- *Chapter 8: Efficient collection processing - concentrates on collection processing.*

They focus on general rules to cheaply optimize every-day development. But they also give some Kotlin-specific suggestions on how performance might be optimized in the critical parts of your program. They should also deepen your understanding of thinking about performance in general. 

Please, remember that when there is a tradeoff between readability and performance, you need to answer yourself what is more important in the component you develop. I included some suggestions, but there is no universal answer.