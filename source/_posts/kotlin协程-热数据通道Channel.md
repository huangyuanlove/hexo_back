---
title: kotlin协程-热数据通道Channel
tags: [Kotlin, Android]
date: 2025-12-06 21:07:48
keywords: 协程基本概念,协程的构造,函数的挂起,协程的上下文,协程的拦截器
---

种一颗树的最好时机是十年前，其次是现在。
学习也一样。
跟着霍老师的《深入理解 Kotlin 携程》学习一下协程。

## 直奔主题，认识Channel
Channel实际上就是一个并发安全的队列，它可以用来连接协程，实现不同协程的通信
``` kotlin
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
上述的代码中构造了两个协程producer和consumer，我么呢没有为它们明确制定调度器，所以他们的调度器都是默认的。其中producer中每隔1秒向Channel发送一个整数，而consumer中一致在读取channel来获取数据并打印，显然发送端比接收端更慢，在没有可以读取的值时，receive是挂起的，直到有新元素到达，这么看来receive一定是一个挂起函数，那么send呢？

## Channel的容量
我们查看send方法的声明，发现它也是挂起函数。那么发送端为什么要挂起？上面也提到，Channel实际上就是一个队列，队列中一定存在缓冲区，一旦这个缓冲区满了，一直没有人调用receive并取走元素，send就要挂起，等待接收者取走元素后再写入Channel。

``` kotlin
public fun <E> Channel(capacity: Int = RENDEZVOUS): Channel<E> =
    when (capacity) {
        RENDEZVOUS -> RendezvousChannel()
        UNLIMITED -> LinkedListChannel()
        CONFLATED -> ConflatedChannel()
        BUFFERED -> ArrayChannel(CHANNEL_DEFAULT_CAPACITY)
        else -> ArrayChannel(capacity)
    }
```
我们构造Cahnnel的时候调用了一个名为Channle的函数，但它不是Channel的构造函数。在Kotlin中，经常定义一个顶级函数来伪装成同名类型的构造器，这本质上是工厂函数。这里有一个Int类型的capacity参数，默认值为RENDEZVOUS。
这时候如果不调用receive，send就会一直挂起等待。

`UNLIMITED`比较好理解，没有限制，来者不拒。
`CONFLATED`这个名字可能有迷惑性，字面意思是合并，但实际上这个函数的效果是只保留最后一个元素，也就是说缓冲区只有一个元素大小，每次有新元素到来，都会覆盖掉旧元素。
`BUFFERED`效果类似于ArrayBlockingQueue，接收一个值作为缓冲区容量大小。

### 迭代Channel
我们在发送和读取的时候写了一个`while(true)`的死循环，因为需要不断地进行读写操作。这里我们可以直接获取一个Channel的Iterator
``` kotlin
    val consumer = GlobalScope.launch {
        val iter = channel.iterator()
        while (iter.hasNext()) {
            val element = iter.next()
            println(element)
            delay(1000)
        }
    }
```

其中 iter.hasNext()是挂起函数，在判断是否有下一个元素的时候就需要去Channel中读取元素了。当然也可以`for ... in ..`:
``` kotlin
    val consumer = GlobalScope.launch {
        for(element in channel) {
            println(element)
            delay(1000)
        }
    }
```
### produce和actor
来看两个便捷的构造生产者和消费这的api，