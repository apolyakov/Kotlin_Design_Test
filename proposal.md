# Automatically unwrap expressions of Java's `Optional<T>` type to expressions of type `T?`

* **Type**: Design proposal
* **Authors**: Alexander Polyakov

# Summary

This KEEP introduces following semantics update:

* allow to initialize a property of type `T?` with an expression of type `Optional<T>`;
* allow to assign an expression of Java's `Optional<T>` type to object of type `T?`;
* allow to pass an expression of Java's `Optional<T>` type as an argument to a function with a corresponding `T?`-typed formal parameter.

These changes reduce amount of boilerplate code one is required to write in order to work in a Kotlin manner with Java code that uses `Optional<T>` type.

This KEEP partially addresses the [KT-20781](https://youtrack.jetbrains.com/issue/KT-20781) feature request. [Future advancements](#future-improvements) section mentions other semantics changes that supplement this proposal and can be discussed further.

# Motivation

## Current state

In Kotlin, the type system distinguishes between references that can hold `null` (nullable references) and those that cannot (non-nullable references). This separation is ensured by the presence of [nullable and non-nullable types](https://kotlinlang.org/docs/null-safety.html#nullable-types-and-non-null-types) in the language. Nullable types are the preferred way of expressing an absence of a value in Kotlin.

As opposed to what happens in Kotlin, in Java any reference can hold `null`. Using such references may cause `NullPointerException` at run time. Java 8 introduced the `Optional<T>` type to its standard library as an explicit and safe way to operate on values that might or might not be present. Since `Optional<T>` type is [recommended](https://www.oracle.com/technical-resources/articles/java/java8-optional.html) to deal with an abcense of a value in Java, the amount of code using it is going to continuously increase in both existing and new Java projects. Thus, improving the user experience for Kotlin developers interoperating with `Optional<T>`-typed Java code becomes an actual problem.

Currently, to work with `Optional<T>`-typed Java code in Kotlin one can either explicitly convert such values to Kotlin's nullable references or use `Optional`'s API. The first approach requires an explicit use of Kotlin's standard library APIs like `getOrNull` extension function defined in `kotlin.jvm.optionals` package. The second approach contradicts with how Kotlin deals with nullability and commonly falls back to the first approach at some point.
```java
// Java
public class User {
    ...
    public static Optional<User> getLoggedInUser() {...}
}
```
```kotlin
// Kotlin
import kotlin.jvm.optionals.getOrNull

val loggedInUser: User? = User.getLoggedInUser() // Error
val loggedInUser: User? = User.getLoggedInUser().getOrNull() // OK
```

This proposal aims to reduce the number of explicit calls to Kotlin's standard library APIs, thereby saving the programmer from having to repeatedly write a boilerplate code calling such APIs.

## Use cases

The main use case for the proposal is an initialization of a `T?`-typed property with an expression of type `Optional<T>`.
```kotlin
val loggedInUser: User? = User.getLoggedInUser() // OK
```
A similar use case is an assignment of an expression of type `Optional<T>` to its left-hand side of type `T?`.
```kotlin
val loggedInUser: User?
if (guestMode) {
    loggedInUser = getGuestUser()
} else {
    loggedInUser = User.getLoggedInUser()
}
```
Another use case that may commonly appear is passing an expression of type `Optional<T>` as an argument to a call of Kotlin function with corresponding formal parameter of type `T?`.
```kotlin
fun logoutCurrentUser(loggedInUser: User?) {...}

logoutCurrentUser(User.getLoggedInUser()) // OK
```
The same logic is also applied to `Optional<T>`-typed fields.
```java
// Java
class User {
    public Integer userID;
    public Optional<String> fullName;
    ...
    public static Optional<User> getLoggedInUser() {...}
}
```
```kotlin
// Kotlin
fun checkExistingUser(id: Int, fullName: String?): Boolean {...}

val user: User? = User.getLoggedInUser()
if (user != null) {
    // User's `fullName` field of type `Optional<T>` is passed as
    // an argument while `String?` was expected
    checkExistingUser(user.userID, user.fullName) // OK
}
```

# Design

## Overall design

The idea of this proposal is to add special cases to the semantics of the property initialization, simple assignment operator and function call expression.

It is proposed to add following rule to the semantics of the property initialization.
> If a `T?`-typed property's initializer expression `e` is of `Optional<T>` type, the initializer expression is treated as `e.getOrNull()`

It is proposed to add following rule to the semantics of the simple assignment operator.
> If the left-hand side operand of the simple assignment operator is of type `T?`, and the right-hand side operand `rhs` of the simple assignment operator is of type `Optional<T>`, the right-hand side operand is treated as `rhs.getOrNull()`.

It is proposed to add following rule to the semantics of the function call.
> If an argument `arg` is of type `Optional<T>` and its corresponding formal parameter is declared with `T?` type, the argument is treated as `arg.getOrNull()`.

> Note: changes to function overload resolution are also required. TBD

## Effect on API stability

This proposal has no impact on API stability.

## Effect on ABI stability

This proposal has no impact on ABI stability.

## Design questions and answers

### Why not make `Optional<T>` a subtype of `T?`?

Despite the fact that such solution would achieve the desired effect without the need to add specially processed cases to the semantics, it would have an impact on ABI stability.

In Kotlin, all generic instantiations share the same code at run time. Making the `Optional<T>` type a subtype of `T?` type would require a recompilation of some generic code.

```kotlin
open class A
class B : A()

fun <T : A?> f(t: T) {...}

f<A>(A()) // share the same f's code
f<B>(B()) // share the same f's code

val opt = Optional.of(A())
f<Optional<A>>(opt) // distinct code for this instantiation is required
```

### Why not interpret `Optional<T>` as `T?`?

Assuming that we work with the following Java code from Kotlin,
```java
// Java
class Box {
    public Boolean isEmpty() {...}

    public static Optional<Box> getOptionalBox() {...}
}
```
such changes would lead to a number of problems.
* Safe-call – compiler's inability to determine the exact call target in some cases. 
    ```kotlin
    // Kotlin
    Box.getOptionalBox()?.isEmpty() // which isEmpty to call?
    ```
    In this example the compiler cannot determine which `isEmpty` method it should call since signatures and return types of `Box::isEmpty` and `Optional<T>::isEmpty` are equal.
* Not-null assertion – API compatibility violation. Currently, applying a not-null assertion operator to an expression of `Optional<T>` type does not have effect since platform types are assumed non-nullable by default. Interpreting `Optional<T>` as `T?`, one may expect a `NullPointerException` being thrown as a result of applying a not-null assertion to an expression referring to an empty optional. This changes the behavior of existing Kotlin code that might apply not-null assertion operator to expressions of type `Optional<T>`.

Moreover, there are other tricky consequences related to how Java-Kotlin-Java hierarchies and reflection would work with such a change.

## Alternatives

* Preserving the status quo. The current API of the Kotlin standard library allows to explicitly convert an expression of type `Optional<T>` to an expression of type `T?`. The proposed changes hide function calls that will happen at run time and add exceptional cases to Kotlin's semantics. As a result, Kotlin becomes more complicated.
* Extending the API of Kotlin's standard library to reduce number of cases where conversion from `Optional<T>` to `T?` is needed. First of all, it creates a separate Kotlin dialect to work with objects that might not have a value. In addition to that, there may be cases for which the conversion is still required by the program's logic.
* Implement convenience features via build-time code generation / IDE support / compiler plugin / etc. While in some rare cases these tools are OK to use in Kotlin, e.g., when the added feature has a complex and/or intrusive implementation such as Jetpack Compose, in general, they are “not the Kotlin way” of adding language features.

# IDE support

In scope of this feature, we also propose to add or extend the following IDE inspections.

* New inspection which recommends to remove an explicit call to `getOrNull` extension method.

# Future advancements

* In this document we consider `Optional<T>` and `T?` only. Further proposals may also consider `Optional<U>` and `T?`, where `U <: T`.
* Considering an automatic wrapping of expressions of type `T?` to expressions of type `Optional<T>`.

# Open questions

* Similar features in other languages.
* Effect on Java-Kotlin-Java hierarchies.
