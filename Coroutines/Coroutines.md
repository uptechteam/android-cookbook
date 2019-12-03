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
