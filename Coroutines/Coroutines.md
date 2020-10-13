# Error handling
Coroutine builders come in two flavours: 

- Propagating exceptions automatically ([launch](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/launch.html) and [actor](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/actor.html))
- Exposing exceptions to user ([async](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/async.html) and [produce](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/produce.html))

The former treat exceptions as unhandled similar to [Thread.defaultUncaughtExceptionHandler](https://docs.oracle.com/javase/8/docs/api/java/lang/Thread.html#setUncaughtExceptionHandler(java.lang.Thread.UncaughtExceptionHandler))

It can be demonstrated by a simple example that creates coroutines in the [GlobalScope](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-global-scope/index.html):
```kotlin
    fun `test_no_exception_handler_in_global_scope`() = runBlocking {
        val launch = GlobalScope.launch {
            println("inside launch, throwing an exception")
            throw IllegalArgumentException()
        }
        val async = GlobalScope.async {
            println("inside async, throwing an exception")
            throw NumberFormatException()
        }
        launch.join()
        println("Joined failed job")
        try {
            async.await()
        } catch (ex: NumberFormatException) {
            println("Caught async exception $ex")
        }
    }
```
Output would be similar to this:

    inside launch, throwing an exception
    inside async, throwing an exception
    Exception in thread "DefaultDispatcher-worker-1" java.lang.IllegalArgumentException
    	at test.basics.ChapterExamplesKt$test_no_exception_handler_in_global_scope$1$launch$1.invokeSuspend(ChapterExamples.kt:34)
    	at .....
    Joined failed job
    Caught async exception java.lang.NumberFormatException

>The reason we use `GlobalScope` here is because according to coroutines architecture `runBlocking` main coroutine is going to be always cancelled when its child completes with exception and we wouldn't be able to run code following `launch.join().` 

## CoroutineExceptionHandler

But what if we want to react to exceptions? [CoroutineExceptionHandler](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-exception-handler/index.html) context element is used as a generic `catch` block of coroutine where custom logging or exception handling may take place. It is similar to using [Thread.uncaughtExceptionHandler](https://docs.oracle.com/javase/8/docs/api/java/lang/Thread.html#setUncaughtExceptionHandler(java.lang.Thread.UncaughtExceptionHandler)).

On JVM it is possible to redefine global exception handler for all coroutines by registering [CoroutineExceptionHandler](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-exception-handler/index.html) via [ServiceLoader](https://docs.oracle.com/javase/8/docs/api/java/util/ServiceLoader.html). Global exception handler is similar to [Thread.defaultUncaughtExceptionHandler](https://docs.oracle.com/javase/8/docs/api/java/lang/Thread.html#setDefaultUncaughtExceptionHandler(java.lang.Thread.UncaughtExceptionHandler)) which is used when no more specific handlers are registered. On Android, `uncaughtExceptionPreHandler` is installed as a global coroutine exception handler.

[CoroutineExceptionHandler](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-exception-handler/index.html) is invoked only on **exceptions** which **are not expected** to be handled by the user, so registering it in the [async](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/async.html) builder and the like of it has no effect.
```kotlin
    private val defaultCoroutineExceptionHandler = CoroutineExceptionHandler { coroutineContext, throwable ->
        println("In a coroutine exception handler, caught $throwable")
    }
    suspend fun `test with exception handler`() = coroutineScope {
        val launch = GlobalScope.launch(defaultCoroutineExceptionHandler) {
            println("Throwing in launch")
            throw IllegalArgumentException()
        }
        val deferred = GlobalScope.async(defaultCoroutineExceptionHandler) { //doesn't work
            println("Throwing in async")
            throw NumberFormatException()
        }
        launch.join()
        deferred.await() 
        Unit
    }
```
Output of this will be:
```kotlin
    Throwing in launch
    Throwing in async
    In a coroutine exception handler, caught java.lang.IllegalArgumentException
    Exception in thread "main" java.lang.NumberFormatException
```
### Cancellation and exceptions

Cancellation is tightly bound with exceptions. Coroutines internally use `CancellationException` for cancellation, these exceptions are ignored by all handlers, so they should be used only as the source of additional debug information, which can be obtained by `catch` block. When a coroutine is cancelled using [Job.cancel](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job/cancel.html), it terminates, but it does not cancel its parent.
```kotlin
    fun `test_inner_cancellation`() = runBlocking {
        var c2: Job? = null
        val c1 = launch(Dispatchers.Default) {
            c2 = launch(Dispatchers.Default) {
                try {
                    while (isActive) {
                        delay(10)
                        print("+")
                    }
                } finally {
                    print("Child is cancelled")
                }
            }
    
            while (isActive) {
                delay(10)
                print(".")
            }
        }
    
        delay(100)
        c2?.cancel()
        delay(150)
        c1.cancel()
        println()
    }
```
The output of this code is:
```kotlin
    +.+.+.+.+.+.+.+.Child is cancelled.............
```
If a coroutine encounters an exception other than `CancellationException`, it cancels its parent with that exception. This behaviour **cannot be overridden** and is used to provide stable coroutines hierarchies for [structured concurrency](https://github.com/Kotlin/kotlinx.coroutines/blob/master/docs/composing-suspending-functions.md#structured-concurrency-with-async) which do not depend on [CoroutineExceptionHandler](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-exception-handler/index.html) implementation. The original exception is handled by the parent when **all** its children terminate.
```kotlin
    val handler = CoroutineExceptionHandler { _, exception -> 
        println("Caught $exception") 
    }
    val job = GlobalScope.launch(handler) {
        launch { // the first child
            try {
                delay(Long.MAX_VALUE)
            } finally {
                withContext(NonCancellable) {
                    println("Children are cancelled, but exception is not handled until all children terminate")
                    delay(100)
                    println("The first child finished its non cancellable block")
                }
            }
        }
        launch { // the second child
            delay(10)
            println("Second child throws an exception")
            throw ArithmeticException()
        }
    }
    job.join()
```
The output of this code is:
```kotlin
    Second child throws an exception
    Children are cancelled, but exception is not handled until all children terminate
    The first child finished its non cancellable block
    Caught java.lang.ArithmeticException
```
### Exception suppression

What happens if multiple children of a coroutine throw an exception? The general rule is "the first exception wins", so the first thrown exception is exposed to the handler. Additional exceptions are suppressed.
```kotlin
    val handler = CoroutineExceptionHandler { _, exception ->
            println("Caught $exception with suppressed ${exception.suppressed.contentToString()}")
        }
    val job = GlobalScope.launch(handler) {
        launch {
            try {
                delay(Long.MAX_VALUE)
            } finally {
                throw ArithmeticException()
            }
        }
        launch {
            delay(100)
            throw IllegalArgumentException()
        }
        delay(Long.MAX_VALUE)
    }
    job.join()
```
>This above code will work properly only on JDK7+ that supports suppressed exceptions

Output: 
```kotlin
    Caught java.lang.IllegalArgumentException with suppressed [java.lang.ArithmeticException]
```
### Supervision

A great example is a server process that spawns several children jobs and needs to supervise their execution, tracking their failures and restarting just those children jobs that had failed.
```kotlin
    val job = SupervisorJob()
    CoroutineScope(coroutineContext + job + defaultCoroutineExceptionHandler).apply {
        val first = launch {
            delay(10)
            println("First throws an exception")
            throw IllegalArgumentException()
        }
    
        val second = launch {
            try {
                println("Second goes to sleep")
                delay(Long.MAX_VALUE)
            } finally {
                println("Second cancelled")
            }
        }
    
        first.join()
        println("First is down")
        println("Second is sleeping")
        println("cancelling supervisor")
        job.cancel()
    }
```
Output:
```kotlin
    Second goes to sleep
    First throws an exception
    In a coroutine exception handler, caught java.lang.IllegalArgumentException
    First is down
    Second is sleeping
    cancelling supervisor
    Second cancelled
```
>You may also use `supervisorScope` instead of `coroutineScope` in the previous example for the same purposes. It propagates cancellation only in one direction and cancels all children only if it has failed itself. It also waits for all children before completion just like coroutineScope does.

>Another crucial difference between regular and supervisor jobs is exception handling. Every child **should** handle its exceptions by **itself** via exception handling mechanisms. This difference comes from the fact that child's failure is not propagated to the parent.

# Materials 

* [Project with examples](https://github.com/uptechteam/coroutines-examples)
* [KEEP Kotlin Evolution and Enhancement Process](https://github.com/Kotlin/KEEP/blob/master/proposals/coroutines.md)
* [Coroutines ~~advanced~~ tutorial Ray Wenderlich](https://www.raywenderlich.com/2117501-kotlin-coroutines-tutorial-for-android-advanced)
* [Coroutines Github Repository](https://github.com/Kotlin/kotlinx.coroutines)
* [Kotlin Coroutines official guide](https://kotlinlang.org/docs/reference/coroutines/coroutines-guide.html)
* [kotlinx.coroutines reference documentation](https://kotlin.github.io/kotlinx.coroutines/)
* [Coroutine context and scope by Roman Elizarov](https://medium.com/@elizarov/coroutine-context-and-scope-c8b255d59055#8293)
* [Reason to avoid `GlobalScope` by Roman Elizarov](https://medium.com/@elizarov/the-reason-to-avoid-globalscope-835337445abc)
* [Explicit concurency by Roman Elizarov](https://medium.com/@elizarov/explicit-concurrency-67a8e8fd9b25)
* [Blocking threads, suspending coroutines by Roman Elizarov](https://medium.com/@elizarov/blocking-threads-suspending-coroutines-d33e11bf4761)
