---
title: kotlin协程-协程框架开发初探-初步实现
tags: [Kotlin, Android]
date: 2025-11-25 09:29:24
toc: true
keywords: 协程基本概念,协程的构造,函数的挂起,协程的上下文,协程的拦截器
---

种一颗树的最好时机是十年前，其次是现在。
学习也一样。
跟着霍老师的《深入理解 Kotlin 携程》学习一下协程。

## 实现一个 delay 函数

这个函数有两个要求：

1. 不需要阻塞线程
2. 是个挂起函数，等待一段时间后可以恢复执行

在 JVM 上，我们可以使用`ScheduledExecutorService`来实现定时执行。

```Kotlin
private val executor = Executors.newScheduledThreadPool(1){
   runnable -> Thread(runnable, "scheduled").apply{isDaemon = true}
}

suspend fun delay(time: Long, unit: TimeUnit = TimeUnit.MILLISECONDS) {
    if (time <= 0) {
        return
    }
    suspendCoroutine<Unit> { continuation ->
        executor.schedule({continuation.resume(Unit)}, time, unit)
    }
}
```

实际上`ScheduledExecutorService`在等待延时事件的时候也会存在对后台线程的阻塞，这难道不是对线程资源的浪费吗？
这里有两个原因：如果当前线程有特殊地位，如某些平台上的 UI 线程 是不能阻塞的。另外一个原因就是一个后台线程可以承载非常多的延时任务，有 10 个协程调用 delay,那么我们字需要阻塞一个后台线程就可以实现这 10 个协程的延时执行。

## 协程的描述
