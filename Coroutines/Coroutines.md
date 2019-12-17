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
