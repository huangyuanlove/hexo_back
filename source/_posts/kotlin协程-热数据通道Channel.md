---
title: kotlin协程-热数据通道Channel
tags: [Kotlin, Android]
date: 2025-12-06 21:07:48
keywords: 协程基本概念,协程的构造,函数的挂起,协程的上下文,协程的拦截器
---

种一颗树的最好时机是十年前，其次是现在。
学习也一样。
跟着霍老师的《深入理解 Kotlin 携程》学习一下协程。

## 直奔主题，认识 Channel

Channel 实际上就是一个并发安全的队列，它可以用来连接协程，实现不同协程的通信

```kotlin
suspend fun main() {
    val channel = Channel<Int>()
    val producer = GlobalScope.launch {
        var i = 0
        while (true) {
            delay(1000)
            channel.send(i++)
        }
    }
    val consumer = GlobalScope.launch {
        while (true) {
            val element = channel.receive()
            println(element)
        }
    }
    producer.join()
    consumer.join()
}
```

上述的代码中构造了两个协程 producer 和 consumer，我么呢没有为它们明确制定调度器，所以他们的调度器都是默认的。其中 producer 中每隔 1 秒向 Channel 发送一个整数，而 consumer 中一致在读取 channel 来获取数据并打印，显然发送端比接收端更慢，在没有可以读取的值时，receive 是挂起的，直到有新元素到达，这么看来 receive 一定是一个挂起函数，那么 send 呢？

## Channel 的容量

我们查看 send 方法的声明，发现它也是挂起函数。那么发送端为什么要挂起？上面也提到，Channel 实际上就是一个队列，队列中一定存在缓冲区，一旦这个缓冲区满了，一直没有人调用 receive 并取走元素，send 就要挂起，等待接收者取走元素后再写入 Channel。

```kotlin
public fun <E> Channel(capacity: Int = RENDEZVOUS): Channel<E> =
    when (capacity) {
        RENDEZVOUS -> RendezvousChannel()
        UNLIMITED -> LinkedListChannel()
        CONFLATED -> ConflatedChannel()
        BUFFERED -> ArrayChannel(CHANNEL_DEFAULT_CAPACITY)
        else -> ArrayChannel(capacity)
    }
```

我们构造 Cahnnel 的时候调用了一个名为 Channle 的函数，但它不是 Channel 的构造函数。在 Kotlin 中，经常定义一个顶级函数来伪装成同名类型的构造器，这本质上是工厂函数。这里有一个 Int 类型的 capacity 参数，默认值为 RENDEZVOUS。
这时候如果不调用 receive，send 就会一直挂起等待。

`UNLIMITED`比较好理解，没有限制，来者不拒。
`CONFLATED`这个名字可能有迷惑性，字面意思是合并，但实际上这个函数的效果是只保留最后一个元素，也就是说缓冲区只有一个元素大小，每次有新元素到来，都会覆盖掉旧元素。
`BUFFERED`效果类似于 ArrayBlockingQueue，接收一个值作为缓冲区容量大小。

### 迭代 Channel

我们在发送和读取的时候写了一个`while(true)`的死循环，因为需要不断地进行读写操作。这里我们可以直接获取一个 Channel 的 Iterator

```kotlin
    val consumer = GlobalScope.launch {
        val iter = channel.iterator()
        while (iter.hasNext()) {
            val element = iter.next()
            println(element)
            delay(1000)
        }
    }
```

其中 iter.hasNext()是挂起函数，在判断是否有下一个元素的时候就需要去 Channel 中读取元素了。当然也可以`for ... in ..`:

```kotlin
    val consumer = GlobalScope.launch {
        for(element in channel) {
            println(element)
            delay(1000)
        }
    }
```

### produce 和 actor

来看两个便捷的构造生产者和消费这的 api，我们可以通过`produce`来启动移动生产者协程，并返回一个`ReceiveChannel`,其他协程就可以通过这个 channel 来获取数据了。  
同样的，我们可以通过`actor`来启动消费者协程，并返回一个`SendChannel`,其他协程就可以通过这个 channel 来发送数据了。

```kotlin
val receiveChannel : ReceiveChannel<Int> = GlobalScope.produce{
    repeat(100){

        delay(100)
        send(it)
    }
}

val sendChannel : SendChannel<Int> = GlobalScope.actor {
    while(true){
        val element = receive()
        println(element)
    }
}

```

## Channel 的关闭

以上面的`produce`方法为例,我们可以看到最终返回的是一个`ProducerCoroutine`,它的定义如下：

```kotlin
private class ProducerCoroutine<E>(
    parentContext: CoroutineContext, channel: Channel<E>
) : ChannelCoroutine<E>(parentContext, channel, true, active = true), ProducerScope<E> {
    override val isActive: Boolean
        get() = super.isActive

    override fun onCompleted(value: Unit) {
        _channel.close()
    }

    override fun onCancelled(cause: Throwable, handled: Boolean) {
        val processed = _channel.close(cause)
        if (!processed && !handled) handleCoroutineException(context, cause)
    }
}

```

我们发现在它的`完成`和`取消`方法中都会调用`_channel.close`的方法。也正是这样，Channel 才被称为热数据流。这里有一点需要注意：对千一个`Channe`如果我们调用了它的`close`方法，它会**立即停止接收**新元素，也就是说这时候它的`isClosedForSend`会立即返回`true`,而由`Channel缓冲区`的存在， 这时候可能还有一些元素没有被处理完，因此要等所有的元素都被读取之后`isClosedForReceive`才会返回`true`.

## BroadcastChannel

在实际环境中，经常会出现一个发送对应多个接收的情况。这里我们就需要 BroadcastChannel 了。

```kotlin
    val broadcastChannel =  BroadcastChannel<Int>(Channel.BUFFERED)

    val producer = GlobalScope.launch {
        List(3){
            delay(1000)
            broadcastChannel.send(it)
        }
    }
    List(3){
        index ->
        GlobalScope.launch {
            val receiveChannel = broadcastChannel.openSubscription()
            for(i in receiveChannel){
                println("[#$index] received $i")
            }
        }
    }.joinAll()
```

这里有个细节需要注意一下，如果把发送端的`dealy(100)`去掉，可能会出现部分元素收不到或者完全收不到的情况，这是因为`BroadcastChannel`在发送的时候没有订阅者，这条消息就被丢弃了。  
我们也可以通过普通的 Channel 进行转换：

```kotlin
val channel = Channel<Int>()
channel.broadcast(3)
```

这里需要注意一下，**BroadcastChannel**被标记为过时了，可以使用`SharedFlow`和`StateFlow`代替。`channel.broadcast()`方法也被标记为过时，也是使用`SharedFlow`来代替

---

以上
