![image](https://miro.medium.com/max/1994/1*OEX52nKgM1SHGO4l1mvV1A.gif)
# What are the coroutines? Basics

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

    interface Continuation<in T> {
    		val context: CoroutineContext
    		fun resumeWith(result: Result<T>)
    }

1. Represents state of coroutine at suspension point.
2. Can be resumed using completion callback used to report success or failure of coroutine.
3. Can be resumed only once

In the snippet below, an existing asynchronous API service that uses callbacks is wrapped into a suspending function, and it propagates the result or error using a Continuation. It’s just an example function, but the idea is there.

    suspend fun <Data, Result> suspendAsyncApi(data: Data): Result =
      suspendCancellableCoroutine { continuation ->
        apiService.doAsyncStuff<Data, Result>(data,
            { result -> continuation.resume(result) }, // resume with a result
            { error -> continuation.resumeWithException(error) } // resume with an error
        )
      }

## CoroutineContext

Coroutines always execute in some context represented by a value of the [CoroutineContext](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/-coroutine-context/) type, defined in the Kotlin standard library.

- Is a set of elements (threading policy, logging, security and transaction aspects of the coroutine execution, coroutine identity and name, etc)
- Immutable
- Can be composed  (see [Coroutine Context](https://www.notion.so/1d06521b-f797-48ed-ba8f-a2397b87660b)  chapter)

Conceptually, coroutine context is an indexed set of elements, where each element has a unique key. It is a mix between a set and a map. Its elements have keys like in a map, but its keys are directly associated with elements, more like in a set. 

## Coroutine Scope

`CoroutineScope`s confine new coroutines by providing a lifecycle-bound component that binds to a coroutine. Every coroutine builder is an extension function defined on the `CoroutineScope` type. `launch()` is an example of a coroutine builder.

Example - `GlobalScope`(Avoid using it). It’s useful for top-level coroutines that operate on the entire app lifetime and aren’t bound to any lifecycle. Typically, you’d use `CoroutineScope` in your `Activity`, `Fragment` or `ViewModel` over `GlobalScope` in an Android app to control when lifecycle events occur.

    class Activity {
        private val mainScope = MainScope()
        
        fun destroy() {
            mainScope.cancel()
        }
    }

Or use delegation with default factory functions:

    class Activity : CoroutineScope by CoroutineScope(Dispatchers.Default) {}

## Coroutine Dispatcher

Dispatchers determine what thread or thread pool the coroutine uses for execution. The dispatcher can confine a coroutine to a specific thread. It can also dispatch it to a thread pool. Less commonly, it can allow a coroutine to run unconfined, without a specific threading rule, which can be unpredictable (see [Dispatchers](https://www.notion.so/3d142124-e287-4b53-82df-f386c2bd49d0) chapter).

Common Dispatchers:

- [Dispatchers.Main](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-dispatchers/-main.html): This dispatcher confines coroutines to the main thread for UI-driven programs, like Swing, JavaFX, or Android apps. **It’s important to note that this dispatcher doesn’t work without adding an environment-specific Main dispatcher dependency in Gradle or Maven**.
- Use [Dispatchers.Main.immediate](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-main-coroutine-dispatcher/immediate.html) for optimum UI performance on updates.
- [Dispatchers.Default](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-dispatchers/-default.html): This is the default dispatcher used by standard builders. It’s backed by a shared pool of JVM threads. Use this dispatcher for CPU intensive computations.
- [Dispatchers.IO](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-dispatchers/-i-o.html): Use this dispatcher for I/O-intensive blocking tasks that uses a shared pool of threads.
- [Dispatchers.Unconfined](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-dispatchers/-unconfined.html): This dispatcher doesn’t confine coroutines to any specific thread. The coroutine starts the execution in the inherited `CoroutineDispatcher` that called it. But after a suspension ends, it may resume in any other thread.
- Private thread pools can be created with [newSingleThreadContext](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/new-single-thread-context.html) and [newFixedThreadPoolContext](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/new-fixed-thread-pool-context.html).
- An arbitrary [Executor](http://docs.oracle.com/javase/8/docs/api/java/util/concurrent/Executor.html) can be converted to dispatcher with [asCoroutineDispatcher](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/java.util.concurrent.-executor/as-coroutine-dispatcher.html) extension function.
