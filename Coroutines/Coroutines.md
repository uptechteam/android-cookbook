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

Conceptually, a [close](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-send-channel/close.html) is like sending a special close token to the channel. The iteration stops as soon as this close token is received, so **there is a guarantee** that all previously sent elements before the close are received. 
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

However, the benefit of a pipeline that uses channels is that it can actually use multiple CPU cores if you run it in Dispatchers.Default context.

In practice, pipelines do involve some other suspending invocations (like asynchronous calls to remote services) and these pipelines cannot be built using `sequence`/`iterator`, because they do not allow arbitrary suspension (restricted suspension), unlike `produce`, which is fully asynchronous.

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
