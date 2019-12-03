# Coroutine Context

Every coroutine in Kotlin has a context that is represented by an instance of CoroutineContext interface. Coroutine context is a persistent set of user-defined objects that can be attached to the coroutine. It may include objects responsible for coroutine threading policy, logging, security and transaction aspects of the coroutine execution, coroutine identity and name, etc. 

Think of a coroutine as a light-weight thread. In this case, coroutine context is just like a collection of thread-local variables. The difference is that `thread-local variables are mutable`, while `coroutine context is **immutable.**`
```kotlin
    interface CoroutineContext {
        operator fun <E : Element> get(key: Key<E>): E?
        fun <R> fold(initial: R, operation: (R, Element) -> R): R
        operator fun plus(context: CoroutineContext): CoroutineContext
        fun minusKey(key: Key<*>): CoroutineContext
    
        interface Element : CoroutineContext {
            val key: Key<*>
        }
    
        interface Key<E : Element>
    }
```
A quick overview of core operations available on context:

- Operator [`get`](http://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/-coroutine-context/get.html) provides type-safe access to an element for a given key. It can be used with `[..]` notation as explained in [Kotlin operator overloading](https://kotlinlang.org/docs/reference/operator-overloading.html).
- Function [`fold`](http://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/-coroutine-context/fold.html) works like [`Collection.fold`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/fold.html) extension in the standard library and provides means to iterate all elements in the context.
- Operator [`plus`](http://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/-coroutine-context/plus.html) works like [`Set.plus`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/plus.html) extension in the standard library and returns a combination of two contexts with elements on the right-hand side of plus replacing elements with the same key on the left-hand side.
- Function [`minusKey`](http://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/-coroutine-context/minus-key.html) returns a context that does not contain a specified key.

The standard library does not contain any concrete implementations of the context elements (except [`EmptyCoroutineContext`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/-empty-coroutine-context/index.html)) but has interfaces and abstract classes so that all these aspects can be defined in libraries in a composable way so that aspects from different libraries can coexist peacefully as elements of the same context.

An [`Element`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/-coroutine-context/-element/index.htmlhttps://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/-coroutine-context/-element/index.html) of the coroutine context is a context itself. It is a singleton context with this element only. This enables the creation of composite contexts by taking library definitions of coroutine context elements and joining them with +. For example, if one library defines auth element with user authorization information, and some other library defines threadPool object with some execution context information, then you can use a launch{} coroutine builder with the combined context using `launch(auth + threadPool) {...}` invocation.

When a coroutine is launched in the `CoroutineScope` of another coroutine, it inherits its context via `CoroutineScope.coroutineContext` and the [Job](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job/index.html) of the new coroutine becomes a *child* of the parent coroutine's job. **When** the **parent** coroutine **is** **cancelled**, all **its** **children** **are** recursively **cancelled** too.

However, when [`GlobalScope`](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-global-scope/index.html) is used to launch a coroutine, there is no parent for the job of the new coroutine. It is therefore not tied to the scope it was launched from and operates independently.
```kotlin
    / launch a coroutine to process some kind of incoming request
    val request = launch {
        // it spawns two other jobs, one with GlobalScope
        GlobalScope.launch {
            println("job1: I run in GlobalScope and execute independently!")
            delay(1000)
            println("job1: I am not affected by cancellation of the request")
        }
        // and the other inherits the parent context
        launch {
            delay(100)
            println("job2: I am a child of the request coroutine")
            delay(1000)
            println("job2: I will not execute this line if my parent request is cancelled")
        }
    }
    delay(500)
    request.cancel() // cancel processing of the request
    delay(1000) // delay a second to see what happens
    println("main: Who has survived request cancellation?")
```
The output of code above:

    job1: I run in GlobalScope and execute independently!
    job2: I am a child of the request coroutine
    job1: I am not affected by cancellation of the request
    main: Who has survived request cancellation?

## Implementing own context

All third-party context elements shall extend [`AbstractCoroutineContextElement`](http://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/-abstract-coroutine-context-element/index.html) class that is provided by the standard library (in `kotlin.coroutines` package). The following style is recommended for library defined context elements. The example below shows a hypothetical authorization context element that stores the current user name:
```kotlin
    class AuthUser(val name: String) : AbstractCoroutineContextElement(AuthUser) { 
    	companion object Key : CoroutineContext.Key<AuthUser>
    }
```
The definition of context `Key` as a companion object of the corresponding element class enables fluent access to the corresponding element of the context. Here is a hypothetical implementation of suspending function that needs to check the name of the current user:
```kotlin
    suspend fun doSomething() { 
    	val currentUser = coroutineContext[AuthUser]?.name
    }
```
It uses a top-level [`coroutineContext`](http://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/coroutine-context.html) property that is available in suspending functions to retrieve the context of the current coroutine.

