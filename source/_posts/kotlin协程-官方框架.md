---
title: kotlin协程-官方框架
tags: [Kotlin, Android]
date: 2025-12-03 20:06:57
keywords: 协程基本概念,协程的构造,函数的挂起,协程的上下文,协程的拦截器
---

种一颗树的最好时机是十年前，其次是现在。
学习也一样。
跟着霍老师的《深入理解 Kotlin 携程》学习一下协程。

## 前言
按照书中的顺序学习感觉不是很流程，看到第五章觉得这个顺序不太适合自己，
因此学习的顺序颠倒一下，先看第六章《Kotlin协程的官放框架》，然后是第七章《Kotlin协程在Android上的应用》，最后在学习第五章《Kotlin协程框架开发初探》

## 协程框架概述

### 构成
Kotlin协程的官方框架kotlinx.coroutines是一套独立于标准库之外的以生产为目的的框架，提供了丰富的API来支撑生产环境中一步程序的设计和实现，主要由以下几部分构成
![协程框架的结构](image/kotlin/官方协程框架的结构.png)

* core:框架的核心逻辑，包括创建、调度、取消、channel、flow等
* ui:包括Android，javafx,swing三个库，用于提供各平台的UI调度器和一些特殊的逻辑
* reactive相关：提供对各种响应式编程框架的支持，
  * reactive
  * reactor
  * rx2
* integration：提供与其他框架的异步回调集成
  * jdk8
  * guava
  * slf4j
  * play-services

### 启动模式

启动模式总共四种，
* DEFAULT 
  * 会立即根据上下文调度协程执行，也就是立即开始调度，再调度前如果携程被取消，其将直接进入取消响应状态
* LAZY
  * 只有携程被需要时，包括主动调用协程的start、join或者await登函数时猜会开始调用度，如果调度前就别取消，那么协程将直接进入异常结束状态
* ATOMIC
  * 协程创建后，立即开始调度，协程执行到第一个挂起点之前不响应取消
* UNDISPATCHED
  * 协程创建后立即在当前函数调用栈中执行，直到遇到第一个真正的挂起点

首先要弄清楚立即调度和立即执行的区别。立即调度表示协程的调度器会立即接收到调度指令，但具体执行的时机以及在哪个线程上执行，还需要根据调度器的具体情况而定，也就是谁立即调用到立即执行之间通常会有一段时间。因此我们可以得出一下结论

* DEFAULT虽然是立即调度，但也有可能在执行前被取消
* UNDISPATCHED是立即执行，因此协程一定会执行
* ATOMIC虽然是立即调度，但其调度和执行两个步骤合成了一个，保证调度和执行是原子操作，因此协程也一定会执行
* UNDISPATCHED和ATOMIC虽然都会保证协程一定执行，但在第一个挂起点之前前者运行在协程创建时所在的线程，后者则会调度到制定的调度器所在的线程上执行

### 调度器

官方预置了4个调度器，我们可以通过Dispatchers对象访问他们。

* Default：默认调度器，适合处理后台计算，其是一个CPU密集型任务调度器。
* IO：IO调度器，适合执行IO相关操作，其是一个IO密集型任务调度器。
* Main：UI调度器，根据平台的不同会被初始化为对应的UI线程的调度器。
* Unconfined：“无所谓”调度器，不要求协程执行在特定线程上。



### 协程的取消检查

我们已经知道挂起函数可以通过suspendCancellableCoroutine来相应所在线程的取消状态，我们在设计异步任务时，异步任务的取消响应点可能就在这些挂起点处。但如果没有挂起点呢？比如标准库中提供的扩展函数 InputStream.copyTo,使用Java BIO来完成。
``` kotlin
public fun InputStream.copyTo(out: OutputStream, bufferSize: Int = DEFAULT_BUFFER_SIZE): Long {
    var bytesCopied: Long = 0
    val buffer = ByteArray(bufferSize)
    var bytes = read(buffer)
    while (bytes >= 0) {
        out.write(buffer, 0, bytes) 
        bytesCopied += bytes
        bytes = read(buffer)
    }
    return bytesCopied
}
```
将这段程序放入协程中之后，就会发现协程的取消状态对它没有任何影响。我们首先想到的是在while循环中添加一个对所在协程的isActive的判断。但实际上官方协程框架还提供了yield函数
``` kotlin
public fun InputStream.copyTo(out: OutputStream, bufferSize: Int = DEFAULT_BUFFER_SIZE): Long {
    ...
    while (bytes >= 0) {
        yield()
    ...
    }
    return bytesCopied
}
```
yield函数的作用主要是检查所在线程的状态，如果已经取消，则抛出取消异常予以响应。此外它还会尝试让出线程的执行权，给其他协程提供执行的机会。

### 协程的超时取消

在网络请求中， 我们可能会有一些特定的接口需要在短时间内没有响应就要取消，这个时间比我们通用配置的时间要短，这种情况下我们就可以使用withTimeout来实现
``` kotlin
GlobalScope.launch {
    val user = withTimeout(1000){
        ....
    }
}
```
如果withTimeout的第二个参数block运行超时，那么就会被取消，取消后withTimeout直接抛出取消异常。如果不希望抛出异常，也可以使用withTimeoutOrNull,它的效果是在超时的情况下返回null。

