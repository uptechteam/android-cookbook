![image](https://miro.medium.com/max/1994/1*OEX52nKgM1SHGO4l1mvV1A.gif)
# Table of contents
1. :abcd: [Basics](https://github.com/uptechteam/android-cookbook/blob/chapter/coroutines/Coroutines/1_Basics.md)
2. :microscope: [Coroutine Scope](#coroutine-scope)
3. :family_man_woman_girl_boy: [Coroutine Context](https://github.com/uptechteam/android-cookbook/blob/chapter/coroutines/Coroutines/3_CoroutineContext.md)
4. :twisted_rightwards_arrows: [Dispatchers](https://github.com/uptechteam/android-cookbook/blob/chapter/coroutines/Coroutines/4_Dispatchers.md)
5. :exclamation: [Error handling (CoroutineExceptionHandler)](https://github.com/uptechteam/android-cookbook/blob/chapter/coroutines/Coroutines/5_ErrorHandling.md)
6. :books: [Materials](#materials)

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
It is defined as an extension function on `CoroutineScope` and takes a `CoroutineContext` as a parameter, so it actually takes two coroutine contexts (since a scope is just a reference to a context). What does it do with them? It merges them using plus operator (see [Coroutine Context](https://github.com/uptechteam/android-cookbook/blob/chapter/coroutines/Coroutines/3_CoroutineContext.md) chapter), producing a set-union of their elements, so that the elements in context parameter are taking precedence over the elements from the scope. The resulting context is used to start a new coroutine, but it is not the context of the new coroutine — is the parent context of the new coroutine. The new coroutine creates its own child `Job` instance (using a job from this context as its parent) and defines its child context as a parent context `plus` its job:

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



# :books: Materials: 

* [KEEP Kotlin Evolution and Enhancement Process](https://github.com/Kotlin/KEEP/blob/master/proposals/coroutines.md)
* [Coroutines ~~advanced~~ tutorial Ray Wenderlich](https://www.raywenderlich.com/2117501-kotlin-coroutines-tutorial-for-android-advanced)
* [Coroutines Github Repository](https://github.com/Kotlin/kotlinx.coroutines)
* [Kotlin Coroutines official guide](https://kotlinlang.org/docs/reference/coroutines/coroutines-guide.html)
* [kotlinx.coroutines reference documentation](https://kotlin.github.io/kotlinx.coroutines/)
* [Coroutine context and scope by Roman Elizarov](https://medium.com/@elizarov/coroutine-context-and-scope-c8b255d59055#8293)
* [Reason to avoid `GlobalScope` by Roman Elizarov](https://medium.com/@elizarov/the-reason-to-avoid-globalscope-835337445abc)
* [Explicit concurency by Roman Elizarov](https://medium.com/@elizarov/explicit-concurrency-67a8e8fd9b25)
* [Blocking threads, suspending coroutines by Roman Elizarov](https://medium.com/@elizarov/blocking-threads-suspending-coroutines-d33e11bf4761)
