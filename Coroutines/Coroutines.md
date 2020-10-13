
![image](https://miro.medium.com/max/1994/1*OEX52nKgM1SHGO4l1mvV1A.gif)
# Table of contents
1. :abcd: [Basics](#basics)
2. :microscope: [Coroutine Scope](#coroutine-scope)
3. :family_man_woman_girl_boy: [Coroutine Context](#coroutine-context)
4. :twisted_rightwards_arrows: [Dispatchers](#dispatchers)
5. :handshake: [Channels](#channels)
6. :rocket: [Flows](#flows)
7. :exclamation: [Error handling](#error-handling)
8. :books: [Materials](#materials)


# Flows

Suspending functions asynchronously returns a single value, `sequence` builder allows us to build lazily evaluated collections but how do we build an **asynchronous** lazily evaluated collection? That's where [Flows](https://kotlinlang.org/docs/reference/coroutines/flow.html) come into action! Let's check an example:
```kotlin
    fun `test sequence blocks thread`() = sequence<Int> {
        (1..5).forEach {
            Thread.sleep(200)
            yield(it)
        }
    }
    
    fun main(args: Array<String>) = runBlocking {
        launch {
            `test sequence blocks thread`().forEach { println(it) }
        }
    
        launch {
            (11..15).forEach {
                delay(100)
                println(it)
            }
        }
    
        Unit
    }
```
The output of this code will be:
```kotlin
    1
    2
    3
    4
    5
    11
    12
    13
    14
    15 
```
`sequence` builder's builder closure is blocking, that's why outputs `11..15` come synchronously after sequence. Changing whole implementation to one that's based on [Flows](https://kotlinlang.org/docs/reference/coroutines/flow.html) would solve this "blocking" problem.
```kotlin
    fun `test flow not blocking`() = flow<Int> {
        (1..5).forEach {
            delay(200) //Note, delay is available for use here since Flow scope is not restricted
            emit(it)
        }
    }
    
    fun main(args: Array<String>) = runBlocking {
    		launch {
            `test flow not blocking`().collect { println(it) }
        }
        launch {
            (11..15).forEach {
                delay(100)
                println(it)
            }
        }
    
        Unit
    }
```
And expected output in this case:
```kotlin
    11
    1
    12
    13
    2
    14
    15
    3
    4
    5
```
>Flow specifications:
>- A builder function for [Flow](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/flow.html) type is called [flow](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/flow.html).
>- Code inside the `flow { ... }` builder block can suspend.
>- Values are *emitted* from the flow using [emit](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/-flow-collector/emit.html) function.
>- Values are *collected* from the flow using [collect](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/collect.html) function.

## Flows are cold

Cold means that code in `flow` block, similar to `sequence`, does not run until the flow is collected. Check out an example:
```kotlin
    fun flow(range: IntRange = 1..5) = flow<Int> {
        println("Inside flow builder")
        range.forEach {
            delay(200)
            emit(it)
        }
    }
    fun `test flows are cold`() = runBlocking {
        println("test flows are cold")
        val intFlow = flow(1..2)
        println("collecting")
        intFlow.collect {
            println(it)
        }
    		println("collecting one more time")
        intFlow.collect {
            println(it)
        }
    }
```
And the output:
```kotlin
    test flows are cold
    collecting
    Inside flow builder
    1
    2
    collecting one more time
    Inside flow builder
    1
    2
```
This is a key reason the `flow()` function is not marked with suspend modifier. It returns quickly and does not wait for anything. The flow starts every time it is collected.

## Cancellation

Flow adheres to the general cooperative cancellation of coroutines. It is fully transparent for cancellation. As usual, flow collection can be cancelled when the flow is suspended in a cancellable suspending function (like delay), and cannot be cancelled otherwise.
```kotlin
    fun `test cancel flow`() = runBlocking {
        withTimeoutOrNull(500) {
            val flow = flow() //calling flow function from previous example
            flow.collect {
                println(it)
            }
        }
        println("cancelled")
    }
```
`withTimeoutOrNull` waits timeout (ms) and returns `null` if time exceeded cancelling flow and giving us the output:
```kotlin
    Inside flow builder
    1
    2
    cancelled
```
## Flow operators

If you are familiar with [ReactiveX](http://reactivex.io/) or with Observer-Observable stream pattern it will be simple as that to switch the reactive thinking to flows. Operators are very similar to what you have experienced before. Examples are `map`, `filter`, `reduce`, `scan` etc.

Worth to split operators into two main groups:

1. Intermediate operators
2. Terminal operators

Intermediate operators are cold, just like flows are. A call to such an operator is not a suspending function itself. It works quickly, returning the definition of a new transformed flow.

Terminal operators on flows are suspending functions that start a collection of the flow. The `collect` operator is the most basic one, but there are other terminal operators, which can make it easier:

- Conversion to various collections like [toList](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/to-list.html) and [toSet](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/to-set.html).
- Operators to get the [first](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/first.html) value and to ensure that a flow emits a [single](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/single.html) value.
- Reducing a flow to a value with [reduce](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/reduce.html) and [fold](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/fold.html).

### Transform operator

Among the flow transformation operators, the most general one is called [transform](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/transform.html). It can be used to imitate simple transformations like [map](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/map.html) and [filter](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/filter.html), as well as implement more complex transformations. Using the `transform` operator, we can [emit](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/-flow-collector/emit.html) arbitrary values an arbitrary number of times.
```kotlin
    fun `test flow transform operator`() = runBlocking {
      flow(1..2)
          .transform { int ->
            println("transforming $int")
            emit(2)
          }
          .collect {
            println("collected $it")
          }
    }
```
Output:
```kotlin
    Inside flow builder
    transforming 1
    collected 2
    transforming 2
    collected 2
```
Check out more examples using intermediate/terminal operators in the sample [project](https://github.com/uptechteam/coroutines-examples) 

## Flow context

Collection of a `flow` always happens in the context of the calling coroutine. For example, if there is a foo flow, then the following code runs in the context specified by the author of this code, regardless of the implementation details of the foo flow:
```kotlin
    withContext(context) {
        foo.collect { value ->
            println(value) // run in the specified context 
        }
    }
```
This property of a flow is called *context preservation*

What if we want to do heavy work on background threads and then collect result on the main thread. With simple suspend functions you would use `withContext` to change the context code running in but that won't work with flows:
```kotlin
    fun `test flow change context`() = runBlocking{
      kotlinx.coroutines.flow.flow {
        kotlinx.coroutines.withContext(Dispatchers.Default) {
          emit(1)
          emit(2)
        }
      }.collect {
        println(it)
      }
    }
```
Will produce following error:
```kotlin
    Exception in thread "main" java.lang.IllegalStateException: Flow invariant is violated:
    		Flow was collected in [BlockingCoroutine{Active}@1a89e326, BlockingEventLoop@4fc6e6f4],
    		but emission happened in [DispatchedCoroutine{Active}@28c4d996, DefaultDispatcher].
    		Please refer to 'flow' documentation or use 'flowOn' instead
```
### flowOn

Similar to [subscribeOn](http://reactivex.io/documentation/operators/subscribeon.html) from ReactiveX package, [flowOn](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/flow-on.html) is a function that shall be used to change the context of the flow emission:
```kotlin
    fun `test flow flowOn operator`() = runBlocking {
      println("test flow flowOn operator")
      flow {
        println(Thread.currentThread().name)
        emit(1)
        emit(2)
      }
          .flowOn(Dispatchers.Default)
          .collect {
            println("collected $it on " + Thread.currentThread().name)
          }
    }
    
    //Output
    

The `flowOn` operator creates another coroutine for an upstream flow when it has to change the [CoroutineDispatcher](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-dispatcher/index.html) in its context thus changing default sequential nature of the flow. 
```
## Flow exceptions

Flow collection can complete with an exception when an emitter or code inside the operators throw an exception. There are several ways to handle these exceptions:

- Use `try-catch` block
- Use `catch` operator function

Let's check an example when producer's code fails with an exception:
```kotlin
    fun `test flow exception`() = runBlocking {
      val flow = flow(1..5)
          .onEach { check(it <= 3) }
    
      try {
        flow
            .collect {
              println(it)
            }
      } catch (ex: Throwable) {
        println("Exception $ex")
      }
    }
```
Output:
```kotlin
    Inside flow builder
    1
    2
    3
    Exception java.lang.IllegalStateException: Check failed.
```
As we see, exception was caught and no more values are emitted after that.

### Exception transparency

Flows must be *transparent to exceptions* and it is a violation of the exception transparency to [emit](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/-flow-collector/emit.html) values in the `flow { ... }` builder from inside of a `try/catch` block. This guarantees that a collector throwing an exception can always catch it using `try/catch` as in the previous example.

The emitter can use a [catch](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/catch.html) operator that preserves this exception transparency and allows encapsulation of its exception handling. The body of the `catch` operator can analyze an exception and react to it in different ways depending on which exception was caught:

- Exceptions can be rethrown using `throw`.
- Exceptions can be turned into emission of values using [emit](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/-flow-collector/emit.html) from the body of [catch](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/catch.html).
- Exceptions can be ignored, logged, or processed by some other code.
```kotlin
    fun `test flow exception catch`() = runBlocking {
      println("test flow exception catch")
      flow(1..5)
          .map { check(it <= 3) }
          .catch { println("Exception $it") }
          .collect {
            println(it)
          }
    }
```
The output of the example is the same, even though we do not have `try/catch` around the code anymore.

The catch intermediate operator, honoring exception transparency, catches only upstream exceptions (that is an exception from all the operators above catch, but not below it)

To catch exception "below" in `collect` terminal operator without using `try/catch` we could do following:
```kotlin
    fun `test flow exception catch declarative`() = runBlocking {
      println("test flow exception catch declarative")
      flow(1..5)
          .onEach {
            check(it <= 3) //suppose u had this check in collect operator
            println("$it")
          }
          .catch { println("Exception $it") }
          .collect()
    }
```
## Launching flow

It is easy to use flows to represent asynchronous events that are coming from some source. In this case, we need an analogue of the `addEventListener` function that registers a piece of code with a reaction for incoming events and continues further work. The [onEach](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/on-each.html) operator can serve this role. However, `onEach` is an intermediate operator. We also need a terminal operator to collect the flow. Otherwise, just calling `onEach` has no effect.

If we use the [collect](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/collect.html) terminal operator after `onEach`, then the code after it will wait until the flow is collected. The [launchIn](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/launch-in.html) terminal operator comes in handy here. By replacing `collect` with `launchIn` we can launch a collection of the flow in a separate coroutine, so that execution of further code immediately continues:
```kotlin
    fun `test flow launch in`() = runBlocking {
      println("test flow launch in")
      flow()
          .onEach { println(it) }
          .launchIn(this)
    
      println("Done")
    }
```
it prints:
```kotlin
    test flow launch in
    Done
    Inside flow builder
    1
    2
    3
    4
    5
```
The required parameter to `launchIn` must specify a [CoroutineScope](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-scope/index.html) in which the coroutine to collect the flow is launched. In the above example this scope comes from the [runBlocking](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/run-blocking.html) coroutine builder, so while the flow is running, this [runBlocking](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/run-blocking.html) scope waits for completion of its child coroutine and keeps the main function from returning and terminating this example.

In actual applications a scope will come from an entity with a limited lifetime. As soon as the lifetime of this entity is terminated the corresponding scope is cancelled, cancelling the collection of the corresponding flow. 

`launchIn` also returns a `Job`, which can be used to cancel the corresponding flow collection coroutine only without cancelling the whole scope or to join it.

Check out more examples in the sample [project](https://github.com/uptechteam/coroutines-examples)

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
