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