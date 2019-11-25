![image](https://miro.medium.com/max/1994/1*OEX52nKgM1SHGO4l1mvV1A.gif)
# Table of contents
1. :abcd: [Basics](#basics)
2. :microscope: [Coroutine Scope](#coroutine-scope)
3. :family_man_woman_girl_boy: [Coroutine Context](#coroutine-context)
4. :twisted_rightwards_arrows: [Dispatchers](#dispatchers)
5. :handshake:[Channels](#channels)
6. [Flows](#flows)
7. :exclamation: [Error handling (CoroutineExceptionHandler)](#error-handling)
8. :books: [Materials](#materials)


# Basics

Coroutine — is an *instance* of *suspendable computation*. It is similar to a thread, in the sense that it takes a block of code to run and has a similar lifecycle — it is *created* and *started*, but it is not bound to any particular thread. It may *suspend* its execution in one thread and *resume* in another one. Moreover, like a future or promise, it may *complete* with some result (which is either a value or an exception).

## **Suspending function:**

**(Function example here)**

1. Has `suspend` modifier.
2. Doesn’t block the thread
3. Can invoke other suspending functions
4. Can be invoked from other suspending functions or lambdas

A modifier `suspend` may be used on any function: top-level function, extension function, member function, local function, or operator function.

>Property getters and setters, constructors, and some operators functions (namely `getValue`, `setValue`, `provideDelegate`, `get`, `set`, and `equals`) cannot have `suspend` modifier. These restrictions may be lifted in the future.

## **Suspending lambda:**

**(Lambda example, launch, future, sequence, runBlocking)**

1. Similar to ordinary [lambda expression](https://kotlinlang.org/docs/reference/lambdas.html)
2. Doesn’t block the thread
3. Can invoke other suspending functions

## **Coroutine builder:**

**(Coroutine builder example: launch, async, produce, runBlocking, sequence, etc..)**

1. Takes *suspending lambda* as an argument
2. Creates coroutine
3. Gives access to coroutine result (Optional)

Examples: launch, async, produce, runBlocking (provide links)

## **Suspension point:**

**(Screenshot of suspension point)**

1. Happens during execution
2. Syntactically - invocation of suspending function

## **Continuation:**
```kotlin
    interface Continuation<in T> {
    		val context: CoroutineContext
    		fun resumeWith(result: Result<T>)
    }
```
1. Represents state of coroutine at suspension point.
2. Can be resumed using completion callback used to report success or failure of coroutine.
3. Can be resumed only once

In the snippet below, an existing asynchronous API service that uses callbacks is wrapped into a suspending function, and it propagates the result or error using a Continuation. It’s just an example function, but the idea is there.
```kotlin
    suspend fun <Data, Result> suspendAsyncApi(data: Data): Result =
      suspendCancellableCoroutine { continuation ->
        apiService.doAsyncStuff<Data, Result>(data,
            { result -> continuation.resume(result) }, // resume with a result
            { error -> continuation.resumeWithException(error) } // resume with an error
        )
      }
```
## CoroutineContext

Coroutines always execute in some context represented by a value of the [CoroutineContext](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/-coroutine-context/) type, defined in the Kotlin standard library.

- Is a set of elements (threading policy, logging, security and transaction aspects of the coroutine execution, coroutine identity and name, etc)
- Immutable
- Can be composed  (see [Coroutine Context](#coroutine-context)  chapter)

Conceptually, coroutine context is an indexed set of elements, where each element has a unique key. It is a mix between a set and a map. Its elements have keys like in a map, but its keys are directly associated with elements, more like in a set. 

## Coroutine Scope

`CoroutineScope`s confine new coroutines by providing a lifecycle-bound component that binds to a coroutine. Every coroutine builder is an extension function defined on the `CoroutineScope` type. `launch()` is an example of a coroutine builder.

Example - `GlobalScope`(Avoid using it). It’s useful for top-level coroutines that operate on the entire app lifetime and aren’t bound to any lifecycle. Typically, you’d use `CoroutineScope` in your `Activity`, `Fragment` or `ViewModel` over `GlobalScope` in an Android app to control when lifecycle events occur.
```kotlin
    class Activity {
        private val mainScope = MainScope()
        
        fun destroy() {
            mainScope.cancel()
        }
    }
```
Or use delegation with default factory functions:
```kotlin
    class Activity : CoroutineScope by CoroutineScope(Dispatchers.Default) {}
```
## Coroutine Dispatcher

Dispatchers determine what thread or thread pool the coroutine uses for execution. The dispatcher can confine a coroutine to a specific thread. It can also dispatch it to a thread pool. Less commonly, it can allow a coroutine to run unconfined, without a specific threading rule, which can be unpredictable (see [Dispatchers](#dispatchers) chapter).

Common Dispatchers:

- [Dispatchers.Main](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-dispatchers/-main.html): This dispatcher confines coroutines to the main thread for UI-driven programs, like Swing, JavaFX, or Android apps. **It’s important to note that this dispatcher doesn’t work without adding an environment-specific Main dispatcher dependency in Gradle or Maven**.
- Use [Dispatchers.Main.immediate](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-main-coroutine-dispatcher/immediate.html) for optimum UI performance on updates.
- [Dispatchers.Default](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-dispatchers/-default.html): This is the default dispatcher used by standard builders. It’s backed by a shared pool of JVM threads. Use this dispatcher for CPU intensive computations.
- [Dispatchers.IO](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-dispatchers/-i-o.html): Use this dispatcher for I/O-intensive blocking tasks that uses a shared pool of threads.
- [Dispatchers.Unconfined](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-dispatchers/-unconfined.html): This dispatcher doesn’t confine coroutines to any specific thread. The coroutine starts the execution in the inherited `CoroutineDispatcher` that called it. But after a suspension ends, it may resume in any other thread.
- Private thread pools can be created with [newSingleThreadContext](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/new-single-thread-context.html) and [newFixedThreadPoolContext](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/new-fixed-thread-pool-context.html).
- An arbitrary [Executor](http://docs.oracle.com/javase/8/docs/api/java/util/concurrent/Executor.html) can be converted to dispatcher with [asCoroutineDispatcher](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/java.util.concurrent.-executor/as-coroutine-dispatcher.html) extension function.

# Coroutine Scope
Taken from [Coroutine Context and Scope](https://medium.com/@elizarov/coroutine-context-and-scope-c8b255d59055#8293).

`CoroutineScope`s confine new coroutines by providing a lifecycle-bound component that binds to a coroutine. Technically it's is an interface that consists of a sole property — `CoroutineContext`. 

It has nothing else but a context. So, why it exists and how is it different from a context itself? The difference between a context and a scope is in their *intended purpose*.

Every coroutine builder is an extension function defined on the `CoroutineScope` type. `Launch` coroutine builder is an example:
```kotlin
    fun CoroutineScope.launch(
        context: CoroutineContext = EmptyCoroutineContext,
        // ...
    ): Job
```
It is defined as an extension function on `CoroutineScope` and takes a `CoroutineContext` as a parameter, so it actually takes two coroutine contexts (since a scope is just a reference to a context). What does it do with them? It merges them using plus operator (see [Coroutine Context](#coroutine-context) chapter), producing a set-union of their elements, so that the elements in context parameter are taking precedence over the elements from the scope. The resulting context is used to start a new coroutine, but it is not the context of the new coroutine — is the parent context of the new coroutine. The new coroutine creates its own child `Job` instance (using a job from this context as its parent) and defines its child context as a parent context `plus` its job:

![](https://miro.medium.com/max/4596/1*zuX5Ozc2TwofXlmDajxpzg.png)


The intended purpose of `CoroutineScope` receiver in the `launch` and in all the other coroutine builders is to reference a scope in which new coroutine is launched. By convention, a context in `CoroutineScope` contains a Job that is going to become a parent of a new coroutine (exception `GlobalScope`).

On the other hand, the intended purpose of context: `CoroutineContext` parameter in `launch` is to provide additional context elements to override elements that would be otherwise inherited from a parent scope. For example:
```kotlin
    launch(CoroutineName("child")) {
        println("My context is $coroutineContext}")        
    }
```
Since the context and the scope are materially the same things, we could have launched a coroutine without having access to the scope and without using `GlobalScope` simply by wrapping the current coroutine context into the instance of `CoroutineScope` as shown in the following function:
```kotlin
    suspend fun doNotDoThis() {
        CoroutineScope(coroutineContext).launch {
            println("I'm confused")
        }
    }
```
**Do not do this!** It makes the scope in which the coroutine is launched opaque and implicit, capturing some outer Job to launch a new coroutine without explicitly announcing it in the function signature.

If you need to launch a coroutine that keeps running after your function returns, then make your function an extension of `CoroutineScope` or pass scope: `CoroutineScope` as a parameter to make your intent clear in your function signature. Do not make these functions suspending:
```kotlin
    fun CoroutineScope.doThis() {
        launch { println("I'm fine") }
    }
    
    fun doThatIn(scope: CoroutineScope) {
        scope.launch { println("I'm fine, too") }
    }
```
### Statements to remember:

- Suspending functions are designed to be non-blocking and should not have side-effects of launching any concurrent work. Suspending functions can and should wait for all their work to complete before returning to the caller.
- Every function that is declared as an extension on CoroutineScope returns immediately, but performs its actions concurrently with the rest of the program. (that's why `returnBlocking` is not an extension function on `CoroutineScope`)

## Global Scope

Great explanation what the `GlobalScope` is and how it affects your coroutine execution by Roman Elizarov: [The reason to avoid GlobalScope](https://medium.com/@elizarov/the-reason-to-avoid-globalscope-835337445abc)

## Key points

- All coroutines are required to be launched in the context of [`CoroutineScope`](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-scope/index.html) or to specify a scope explicitly.
- By default `GlobalScope` uses `Dispatchers.Default` to execute coroutines.
- Coroutines launched using `GlobalScope` don't have parent job so it's developers responsibility to handle coroutine's `Job`. Example:
```kotlin
    val jobs = mutableListOf<Job>()
    for (i in 1..2) {
        jobs += GlobalScope.launch {
            work(i)
        }
    }
    jobs.forEach { it.join() }
```

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

# Dispatchers

The coroutine context includes a [`CoroutineDispatcher`](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-dispatcher/index.html) that determines what thread or threads the corresponding coroutine uses for its execution. The coroutine dispatcher can confine coroutine execution to a specific thread, dispatch it to a thread pool, or let it run unconfined.

All coroutine builders like [`launch`](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/launch.html) and [`async`](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/async.html) accept an optional [Coroutine Context](https://www.notion.so/1d06521b-f797-48ed-ba8f-a2397b87660b) parameter that can be used to explicitly specify the dispatcher for the new coroutine and other context elements.

Try following examples:
```kotlin
    launch { // context of the parent, main runBlocking coroutine
        println("main runBlocking      : I'm working in thread ${Thread.currentThread().name}")
    }
    launch(Dispatchers.Unconfined) { // not confined -- will work with main thread
        println("Unconfined            : I'm working in thread ${Thread.currentThread().name}")
    }
    launch(Dispatchers.Default) { // will get dispatched to DefaultDispatcher 
        println("Default               : I'm working in thread ${Thread.currentThread().name}")
    }
    launch(newSingleThreadContext("MyOwnThread")) { // will get its own new thread
        println("newSingleThreadContext: I'm working in thread ${Thread.currentThread().name}")
    }
```
Output (order may differ):
```
    Unconfined            : I'm working in thread main
    Default               : I'm working in thread DefaultDispatcher-worker-1
    newSingleThreadContext: I'm working in thread MyOwnThread
    main runBlocking      : I'm working in thread main
```
When `launch` is used without parameters, it inherits the context (and thus dispatcher) from the [CoroutineScope](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-scope/index.html) it is being launched from. In this case, it inherits the context of the main `runBlocking` coroutine which runs in the `main` thread.

[Dispatchers.Unconfined](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-dispatchers/-unconfined.html) is a special dispatcher that runs in a thread it was launched from, thus also `main`.

The default dispatcher that is used when coroutines are launched in [GlobalScope](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-global-scope/index.html) is represented by [Dispatchers.Default](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-dispatchers/-default.html) and uses a shared background pool of threads, so `launch(Dispatchers.Default)` uses the same dispatcher as `GlobalScope.launch`.

[newSingleThreadContext](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/new-single-thread-context.html),  [newFixedThreadPoolContext](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/new-fixed-thread-pool-context.html) creates a thread/pool of threads respectively for the coroutine to run. A dedicated thread is a very expensive resource. In a real application, it must be either released, when no longer needed, using the [close](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-executor-coroutine-dispatcher/close.html) function, or stored in a top-level variable and reused throughout the application.

## Unconfined dispatcher

- The [Dispatchers.Unconfined](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-dispatchers/-unconfined.html) coroutine dispatcher starts a coroutine in the **caller** thread
- If suspension happens, resumes in the thread that is fully determined by the suspending function that was invoked.
- Dispatcher is inherited from the outer [CoroutineScope](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-scope/index.html) by default

The unconfined dispatcher is appropriate for coroutines which neither consume CPU time nor update any shared data (like UI) confined to a specific thread. 
```kotlin
    fun main() = runBlocking {
    		launch(Dispatchers.Unconfined) { // not confined -- will work with main thread
    		    println("Unconfined      : I'm working in thread ${Thread.currentThread().name}")
    		    delay(500)
    		    println("Unconfined      : After delay in thread ${Thread.currentThread().name}")
    		}
    		launch { // context of the parent, main runBlocking coroutine
    		    println("main runBlocking: I'm working in thread ${Thread.currentThread().name}")
    		    delay(1000)
    		    println("main runBlocking: After delay in thread ${Thread.currentThread().name}")
    		}
    }
```
Produces the output:
```
    Unconfined      : I'm working in thread main
    main runBlocking: I'm working in thread main
    Unconfined      : After delay in thread kotlinx.coroutines.DefaultExecutor
    main runBlocking: After delay in thread main
```
So, the coroutine with the context inherited from `runBlocking {...}` continues to execute in the `main` thread, while the unconfined one resumes in the default executor thread that the [delay](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/delay.html) function is using.

>The unconfined dispatcher is an advanced mechanism that can be helpful in certain corner cases where dispatching of a coroutine for its execution later is not needed or produces undesirable side-effects, because some operation in a coroutine must be performed right away. The unconfined dispatcher should not be used in general code.

# Channels
Channels provide a way to transfer a stream of values.
Characteristics:

- Non-blocking
- Can be closed
- Can be iterated with for loop (see below)
- Channels respect the order of invocation, working in FIFO style.
- Supports Fan-In, Fan-Out
- Supports Buffering (Rendezvous, Unlimited, Conflated, Fixed)

Example of channel use:
```kotlin
      val channel = Channel<Int>()
    
      launch(Dispatchers.Default) {
        channel.send(1)
        delay(100)
        channel.send(2)
      }
      channel.consumeEach {
        print("$it ")
      }
```
## Closing channels

Channel can be closed to indicate that no more elements are coming. 

Conceptually, a [close](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-send-channel/close.html) is like sending a special close token to the channel. The iteration stops as soon as this close token is received, so **there is a guarantee** that all previously sent elements before the close are received. 
Let's update our previous example and add `close` call:
```kotlin
     val channel = Channel<Int>()
    
      launch(Dispatchers.Default) {
        channel.send(1)
        delay(100)
        channel.send(2)
    		channel.close()
      }
      for(it in channel) { //We can also iterate through channel with for loop until channel is closed
        print("$it ")
      }
    	println("Consumed!")
```
## Produce and Pipelines

There is a convenient coroutine builder named `produce` that makes it easy to construct the channel:
```kotlin
    fun CoroutineScope.produceNumbers() = produce {
      repeat(10) {
        send(it + 1)
        delay(400L)
      }
    }
```
A pipeline is a pattern where one coroutine is producing, possibly infinite, stream of values.

And another coroutine or coroutines are consuming that stream, doing some processing, and producing some other results:
```kotlin
    fun CoroutineScope.map(channel: ReceiveChannel<Int>, mapper : (Int) -> Int) = produce<Int> {
      for(value in channel) send(mapper(value))
    }
```
Main code:
```kotlin
    val producer = produceNumbers()
    val mapper = map(producer) {
       it * 2
    }
    for (value in mapper) {
      println("$value")
    }
    print("Done!")
```
There are variety of methods like `map`, `filter` etc in the standard library available to use with `Channels`. 

All channel operators are **deprecated** now in favor of Flow and will be removed in 1.4

You can also use an `iterator` extension function from the standard library to replace channels in a pipeline above. 

However, the benefit of a pipeline that uses channels is that it can actually use multiple CPU cores if you run it in Dispatchers.Default context.

In practice, pipelines do involve some other suspending invocations (like asynchronous calls to remote services) and these pipelines cannot be built using `sequence`/`iterator`, because they do not allow arbitrary suspension (restricted suspension), unlike `produce`, which is fully asynchronous.

## Fan-In & Fan-Out

Multiple coroutines can send to and receive from channel distributing the work between themselves. 

### Fan-Out

Let's use our `produceNumbers` function as a generator and distribute the numbers between multiple consumers:
```kotlin
    suspend fun numberProcessor(id : String, channel : ReceiveChannel<Int>) {
      for (msg in channel) {
        println("Processor #$id received $msg")
      }
    }
    
    val channel = produceNumbers()
    launch { numberProcessor("1", channel) }
    launch { numberProcessor("2", channel) }
    launch { numberProcessor("3", channel) }
    
    launch {
    	delay(2000L)
    	channel.cancel()
    }
```
The output will be similar to the the following one, albeit the processor ids that receive each specific integer may be different:
```kotlin
    Processor #1 received 1
    Processor #1 received 2
    Processor #2 received 3
    Processor #3 received 4
    Processor #1 received 5
    Processor #2 received 6
    Processor #3 received 7
    Processor #1 received 8
    Processor #2 received 9
    Processor #3 received 10
```
Note that cancelling a producer coroutine closes its channel, thus eventually terminating iteration over the channel that processor coroutines are doing.

Also, pay attention to how we explicitly iterate over the channel with `for` loop to perform fan-out in `launchConsumer` code. Unlike `consumeEach`, this `for` loop pattern is perfectly safe to use from multiple coroutines. If one of the processor coroutines fails, then others would still be processing the channel, while a processor that is written via `consumeEach` always consumes (cancels) the underlying channel on its normal or abnormal completion.  (see source code below)
```kotlin
    public inline fun <E, R> ReceiveChannel<E>.consume(block: ReceiveChannel<E>.() -> R): R {
        var cause: Throwable? = null
        try {
            return block()
        } catch (e: Throwable) {
            cause = e
            throw e
        } finally {
            cancelConsumed(cause) //as you may see whole channel is being closed when exception rises
        }
    }
```
### Fan-In

Function that sends to the provided channel after a specified delay until current coroutine context is active and channel not closed.
```kotlin
    suspend fun sendToChannel(channel: SendChannel<Any>, value: Any, delay: Long) {
        while (coroutineContext.isActive && !channel.isClosedForSend) {
            delay(delay)
            if (!channel.isClosedForSend)
                channel.send(value)
        }
    }

    //And main code
    val channel = Channel<Any>()
    //Launching 3 coroutines 
    launch { sendToChannel(channel, "ONE", 200L) }
    launch { sendToChannel(channel, "TWO", 400L) }
    launch { sendToChannel(channel, 1, 500L) }
    //Launching one more guy to close the channel in 3 secs.
    launch {
    	delay(3000L)
      channel.close()
    }
    //Printing values while they are being delivered by coroutines
    for (value in channel) {
        println(value)
    }
    //Done!
    print("Done!")
```
Output:
```kotlin
    ONE
    TWO
    ONE
    1
    ONE
    TWO
    ONE
    ....
    ....
    ONE
    ONE
    TWO
    ONE
    1
    ONE
    TWO
    ONE
    Done!
```
As u may see, we were able to send data to shared channel from multiple coroutines 

## Buffering

The channels shown so far had no buffer. Unbuffered channels transfer elements when sender and receiver meet each other (aka rendezvous). If `send` is invoked first, then it is suspended until `receive` is invoked, if `receive` is invoked first, it is suspended until `send` is invoked.

Both [Channel()](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-channel/index.html) factory function and produce builder take an optional capacity parameter to specify buffer size. 
Types of buffered channels:

- **Rendezvous** - This channel does not have any buffer at all (aka capacity = 0).
- **Unlimited** - This is a channel with linked-list buffer of an unlimited capacity (limited only by available memory). Sender to this channel never suspends and `offer` always returns `true`.
- **Conflated** - This channel buffers at most one element and conflates all subsequent send and offer invocations, so that the receiver always gets the most recently sent element. Sender to this channel never suspends and `offer` always returns `true` (BehaviorSubject, BehaviorRelay style).
- **Fixed** - Fixed array based channel.

# Flows

# Error handling

# Materials 

* [KEEP Kotlin Evolution and Enhancement Process](https://github.com/Kotlin/KEEP/blob/master/proposals/coroutines.md)
* [Coroutines ~~advanced~~ tutorial Ray Wenderlich](https://www.raywenderlich.com/2117501-kotlin-coroutines-tutorial-for-android-advanced)
* [Coroutines Github Repository](https://github.com/Kotlin/kotlinx.coroutines)
* [Kotlin Coroutines official guide](https://kotlinlang.org/docs/reference/coroutines/coroutines-guide.html)
* [kotlinx.coroutines reference documentation](https://kotlin.github.io/kotlinx.coroutines/)
* [Coroutine context and scope by Roman Elizarov](https://medium.com/@elizarov/coroutine-context-and-scope-c8b255d59055#8293)
* [Reason to avoid `GlobalScope` by Roman Elizarov](https://medium.com/@elizarov/the-reason-to-avoid-globalscope-835337445abc)
* [Explicit concurency by Roman Elizarov](https://medium.com/@elizarov/explicit-concurrency-67a8e8fd9b25)
* [Blocking threads, suspending coroutines by Roman Elizarov](https://medium.com/@elizarov/blocking-threads-suspending-coroutines-d33e11bf4761)
