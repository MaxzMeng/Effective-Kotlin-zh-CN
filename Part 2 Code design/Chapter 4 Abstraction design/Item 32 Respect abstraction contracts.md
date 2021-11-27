## Item 32: Respect abstraction contracts

Both contract and visibility are kind of an agreement between developers. This agreement nearly always can be violated by a user. Technically, everything in a single project can be hacked. For instance, it is possible to use reflection to open and use anything we want:

``` kotlin
class Employee {
   private val id: Int = 2
   override fun toString() = "User(id=$id)"

   private fun privateFunction() {
       println("Private function called")
   }
}

fun callPrivateFunction(employee: Employee) {
   employee::class.declaredMemberFunctions
        .first { it.name == "privateFunction" }
        .apply { isAccessible = true }
        .call(employee)
}

fun changeEmployeeId(employee: Employee, newId: Int) {
   employee::class.java.getDeclaredField("id")
        .apply { isAccessible = true }
        .set(employee, newId)
}

fun main() {
   val employee = Employee()
   callPrivateFunction(employee) 
   // Prints: Private function called

   changeEmployeeId(employee, 1)
   print(employee) // Prints: User(id=1)
}
```

Just because you can do something, doesn’t mean that it is fine to do it. Here we very strongly depend on the implementation details like the names of the private property and the private function. They are not part of a contract at all, and so they might change at any moment. This is like a ticking bomb for our program. 

Remember that a contract is like a warranty. As long as you use your computer correctly, the warranty protects you. When you open your computer and start hacking it, you lose your warranty. The same principle applies here: when you break the contract, it is your problem when implementation changes and your code stops working. 

### Contracts are inherited

It is especially important to respect contracts when we inherit from classes, or when we extend interfaces from another library. Remember that your object should respect their contracts. For instance, every class extends `Any` that have `equals`and `hashCode` methods. They both have well-established contracts that we need to respect. If we don’t, our objects might not work correctly. For instance, when `hashCode` is not consistent with `equals`, our object might not behave correctly on `HashSet`. Below behavior is incorrect because a set should not allow duplicates:

``` kotlin
class Id(val id: Int) {
   override fun equals(other: Any?) =
       other is Id && other.id == id
}

val mutableSet = mutableSetOf(Id(1))
mutableSet.add(Id(1))
mutableSet.add(Id(1))
print(mutableSet.size) // 3
```

In this case, it is that `hashCode` do not have implementation consistent with `equals`. We will discuss some important Kotlin contracts in *Chapter 6: Class design*. For now, remember to check the expectations on functions you override, and respect those. 

### Summary

If you want your programs to be stable, respect contracts. If you are forced to break them, document this fact well. Such information will be very helpful to whoever will maintain your code. Maybe that will be you, in a few years’ time.