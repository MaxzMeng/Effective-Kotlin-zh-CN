The null-safety introduced by Kotlin is amazing. Java was known in the community from Null-Pointer Exceptions (NPE), and Kotlin’s safety mechanisms make them rare or eliminate them entirely. Although one thing that cannot be secured completely is a connection between a language that does not have solid null-safety - like Java or C - and Kotlin. Imagine that we use a Java method that declares String as a return type. What type should it be in Kotlin?

If it is annotated with the @Nullable annotation then we assume that it is nullable and we interpret it as a String?. If it is annotated with @NotNull then we trust this annotation and we type it as String. Though, what if this return type is not annotated with either of those annotations?

```java
// Java 
public class JavaTest {
    public String giveName() { 
      	// ...
    }
}
```

We might say that then we should treat such a type as nullable. This would be a safe approach since in Java everything is nullable. However, we often know that something is not null so we would end up using the not-null assertion !! in many places all around our code.

The real problem would be when we would need to take generic types from Java. Imagine that a Java API returns a List<User> that is not annotated at all. If Kotlin would assume nullable types by default, and we would know that this list and those users are not null, we would need to not only assert the whole list but also filter nulls:

```java
// Java 
public class UserRepo {
		public List<User> getUsers() { 
      	//*** 
    }
}

// Kotlin 
val users: List<User> = UserRepo().users!!.filterNotNull()
```

What if a function would return a List<List<User>> instead?

Gets complicated:

```kotlin
val users: List<List<User>> = UserRepo().groupedUsers!!.map { it!!.filterNotNull() }
```

Lists at least have functions like map and filterNotNull. In other

generic types, nullability would be an even bigger problem. This is why instead of being treated as nullable by default, a type that comes from Java and has unknown nullability is a special type in Kotlin. It is called a platform type.

Platform type - a type that comes from another language and has unknown nullability.

Platform types are notated with a single exclamation mark ! after the type name, such as String!. Though this notation cannot be used in a code. Platform types are non-denotable, meaning that one cannot write them down explicitly in the language. When a platform value is assigned to a Kotlin variable or property, it can be inferred but it cannot be explicitly set. Instead, we can choose the type that we expect: Either a nullable or a non-null type.

```kotlin
// Java 
public class UserRepo {
    public User getUser() {
    		//...
    } 
}

// Kotlin 
val repo = UserRepo() 
val user1 = repo.user // Type of user1 is User! 
val user2: User = repo.user // Type of user2 is User 
val user3: User? = repo.user // Type of user3 is User?
```

Thanks to this fact, getting generic types from Java is not problematic:

```kotlin
val users: List<User> = UserRepo().users 
val users: List<List<User>> = UserRepo().groupedUsers
```

The problem is that is still dangerous because something we assumed to be not-null might be null. This is why for safety reasons I always suggest to be very conscientious of when we get platform types from Java. Remember that even if a function does not return null now, that doesn’t mean that it won’t change in the future. If its designer hasn’t specified it by an annotation or by describing it in a comment, they can introduce this behavior without changing any contract.

If you have some control over Java code that needs to interoperate with Kotlin, introduce @Nullable and @NotNull annotations wherever possible.

```java
// Java 
import org.jetbrains.annotations.NotNull; 
public class UserRepo {
		public @NotNull User getUser() {
				//...
    } 
}
```

This is one of the most important steps when we want to support Kotlin developers well (and it’s also important information for Java developers). Annotating many exposed types was one of the most important changes that were introduced into the Android API after Kotlin became a first-class citizen. This made the Android API much more Kotlin-friendly.

Note that many different kinds of annotations are supported, including those by:

- JetBrains (@Nullable and @NotNull from org.jetbrains.annotations)

- Android (@Nullable and @NonNull from androidx.annotation as well as from com.android.annotations and from the support library android.support.annotations)

- JSR-305 (@Nullable,@CheckForNull and @Nonnull from javax.annotation)

- JavaX (@Nullable, @CheckForNull, @Nonnull from javax.annotation)

- FindBugs (@Nullable, @CheckForNull, @PossiblyNull and @NonNull from edu.umd.cs.findbugs.annotations)

- ReactiveX (@Nullable and @NonNull from io.reactivex.annotations)

- Eclipse (@Nullable and @NonNull from org.eclipse.jdt.annotation)

- Lombok (@NonNull from lombok)

Alternatively, you can specify in Java that all types should be Notnull by default using JSR 305’s @ParametersAreNonnullByDefault annotation.

There is something we can do in our Kotlin code as well. My recommendation for safety reasons is to eliminate these platform types as soon as possible. To understand why, think about the difference between how statedType and platformType functions behave in this example:

```kotlin
// Java 
public class JavaClass {
		public String getValue() {
				return null;
    } 
}

// Kotlin 
fun statedType() {

		val value: String = JavaClass().value

		//...

		println(value.length) 
}

fun platformType() { 
  	val value = JavaClass().value 
  	//...
  	println(value.length) 
}
```

In both cases, a developer assumed that getValue will not return null and he or she was wrong. This results in an NPE in both cases, but there’s a difference in where that error happens.

In statedType the NPE will be thrown in the same line where we get the value from Java. It would be absolutely clear that we wrongly assumed a not-null type and we got null. We would just need to change it and adjust the rest of our code to this change.

In platformType the NPE will be thrown when we use this value as not-nullable. Possibly from a middle of some more complex expression. Variable typed as a platform type can be treated both as nullable and not-nullable. Such variable might be used few times safely, and then unsafely and throw NPE then. When we use such properties, typing system do not protect us. It is a similar situation as in Java, but in Koltin we do not expect that we might have NPE just from using an object. It is very likely that sooner or later someone will use it unsafely, and then we will end up with a runtime exception and its cause might be not so easy to find.

```kotlin
// Java 
public class JavaClass {
		public String getValue() {
				return null;
    } 
}

// Kotlin 
fun platformType() {
		val value = JavaClass().value
		//...
		println(value.length) // NPE 
}

fun statedType() { 
  	val value: String = JavaClass().value // NPE 
  	//...
  	println(value.length) 
}
```

What is even more dangerous, platform type might be propagated further. For instance, we might expose a platform type as a part of our interface:

```kotlin
interface UserRepo { 
    fun getUserName() = JavaClass().value 
}
```

In this case, methods inferred type is a platform type. This means that anyone can still decide if it is nullable or not. One might choose to treat it as nullable in a definition site, and as a non-nullable in the use-site:

```kotlin
class RepoImpl: UserRepo {
   override fun getUserName(): String? {
       return null
   }
}

fun main() {
   val repo: UserRepo = RepoImpl()
   val text: String = repo.getUserName() // NPE in runtime
   print("User name length is ${text.length}")
}
```

Propagating a platform type is a recipe for disaster. They are problematic, and for safety reasons, we should always eliminate them as soon as possible. In this case, IDEA IntelliJ helps us with a warning:

![](../../assets/chapter1/chapter1-3.png)

#### Summary

Types that come from another language and has unknown nullability are known as platform types. Since they are dangerous, we should eliminate them as soon as possible, and do not let them propagate. It is also good to specify types using annotations that specify nullability on exposed Java constructors, methods and fields. It is precious information both for Java and Kotlin developers using those elements.