---
title: kotlin协程-基础设施篇-函数的挂起
tags: [Kotlin, Android]
date: 2025-11-24 12:06:38
toc: true
keywords: 协程基本概念,协程的构造,函数的挂起,协程的上下文,协程的拦截器
---

种一颗树的最好时机是十年前，其次是现在。
学习也一样。
跟着霍老师的《深入理解 Kotlin 携程》学习一下协程。

## 函数的挂起

协程的挂起和恢复能力本质上就是函数的挂起和恢复。在 kotlin 中，使用`suspend`关键字修饰的函数叫做`挂起函数`,这种函数只能在协程提或者其他挂起函数中调用。这样我们就可以把 kotlin 中的函数归为两类：普通函数和挂起函数。
挂起函数不一定真的会被挂起，它只是提供了一个挂起的条件。比如我们可以让挂起函数直接 return。

```Kotlin
suspend fun testOne(param:Int):Int{
    return  param * param
}
```

这里只是举个例子，这样写的话编辑器会提示我们`suspend`是个冗余的修饰符。
![冗余的 suspend 修饰符](image/kotlin/redundant_suspend.png)

我们再看另外一个函数

```Kotlin
suspend fun testThree(param: Int) {
    val result = suspendCoroutine<Int> { continuation -> println("continuation is $continuation")
        continuation.resumeWith(Result.success(param))
    }
    println("result is $result")
}

```

可以看到，所谓的协程的挂起，就是程序执行流程发生异步调用时，当前调用流程进入等待状态。

## 挂起点

对比上面两个函数来看，如果一个函数想要让自己挂起，所需要的无非就是一个 Continuation 实例，那么这个实例怎么来的？在前面的文章中也提到过，协程体本身也是一个 Continuation 实例，也正式因为这个原因，挂起函数才能在协程体内运行。
在协程内部，挂起函数的调用处被称为挂起点，挂起点如何发生异步调用，那么当前协程就会被挂起，直到对应的 Continuation 的 resume 函数被调用才会恢复执行。

在上面的`testThree`函数中，从打印结果可以看出获取到的`continuation`对象是一个`SafeContinuation`。

> continuation is SafeContinuation for Continuation at coroutines.MainKt.testThree

这个类的作用也很简单：确保只有发生异步调用的时候才会挂起。
比如下面的函数就不会挂起

```Kotlin
suspend fun notSuspend() = suspendCoroutine<Int> { continuation -> continuation.resumeWith(Result.success(0)) }
```

而异步调用是否发生取决于 resume 函数与对应的挂起函数的调用是否在相同的调用栈上。

## CPS 变换

CPS（Continuation-Passing Style，续延传递风格）是一种编程风格，其核心思想是：函数不直接返回结果，而是接收一个额外的参数，即“续延”（Continuation），并将结果传递给这个续延。
简单来说，续延是一个函数，它代表了一个计算的“未来”或“剩余部分”。它接收一个参数（即当前计算的结果），并利用这个参数来完成后续所有的计算。
举个例子

**直接风格**

```kotlin
result = add(1, 2)
print(result)
```

**续延风格**

```kotlin
add_cps(1, 2, lambda result: print(result))
```

在 kotlin 里面，CPS 变换是通过传递 Continuation 实例来控制异步调用流程的。Kotlin 协程挂起的时候，就是将挂起点的信息保存在了 Continuation 对象中，它携带了协程继续执行时所需要的上下文信息，在恢复执行时，只需要执行它的恢复即可。

## 协程上下文

上下文在很多地方都有它的身影，它只是一个概念，比如 Android 中的上下文，Spring 中的上下文，HarmonyOS 中的上下文。一般情况下，上下文承载了资源管理、获取配置等功能。
那么协程的上下文也是这样，只不过相比于其他的上下文，协程上下文有更显著的数据结构特征。
我们可以使用操作符`+`来对不同类型的协程上下文进行组装，

```kotlin
var coroutineContext: CoroutineContext = EmptyCoroutineContext
coroutineContext += CoroutineName("huangyuan-01")
coroutineContext += CoroutineName("huangyuan-02")
coroutineContext += CoroutineExceptionHandler { context, ex -> println("CoroutineExceptionHandler got $ex") }
suspend {
    println("In Coroutine [${coroutineContext[CoroutineName]} ].")
    println("In Coroutine [${coroutineContext[CoroutineExceptionHandler]} ].")
    5
}.startCoroutine(object : Continuation<Int> {
    override val context = coroutineContext

    override fun resumeWith(result: Result<Int>) {
        result.onFailure {
            context[CoroutineExceptionHandler]?.handleException(context, it)
        }.onSuccess {
            println("Coroutine [${coroutineContext[CoroutineName]}] completed success")
            println("Coroutine [${coroutineContext[CoroutineExceptionHandler]}] completed success")
        }
    }
})
```

我们可以看到打印的日志

> In Coroutine [CoroutineName(huangyuan-02) ].
> In Coroutine [coroutines.MainKt$main$$inlined$CoroutineExceptionHandler$1@4f933fd1 ].
> Coroutine [CoroutineName(huangyuan-02)] completed success
> Coroutine [coroutines.MainKt$main$$inlined$CoroutineExceptionHandler$1@4f933fd1] completed success

### 合并规则

- 非交换性：ctx1 + ctx2 ≠ ctx2 + ctx1（顺序很重要）
- 结合性：(ctx1 + ctx2) + ctx3 = ctx1 + (ctx2 + ctx3)
- 键唯一性：相同 Key 的元素会被覆盖

注意，这里的 Key 定义是

```Kotlin
public interface Key<E : Element>
public interface Element : CoroutineContext {
  public val key: Key<*>
}
```

### CombinedContext

注意这里还有一个比较重要的类,后面重写操作符的时候会用到

```Kotlin
internal class CombinedContext(
    private val left: CoroutineContext,
    private val element: Element
) : CoroutineContext, Serializable {
    override fun <E : Element> get(key: Key<E>): E? {
      var cur = this
      while (true) {
          cur.element[key]?.let { return it }
          val next = cur.left
          if (next is CombinedContext) {
              cur = next
          } else {
              return next[key]
          }
      }
  }
}
```

### 示例

**规则 1：相同 Key 的元素，右边的覆盖左边的**

```Kotlin
val context1 = Job() + CoroutineName("First")
val context2 = CoroutineName("Second") + Dispatchers.IO

val result = context1 + context2
// 结果包含：Job（来自context1）, Dispatchers.IO（来自context2）, CoroutineName("Second")（来自context2，覆盖了context1的）
```

**规则 2：EmptyCoroutineContext 是中性元素**

```Kotlin
val context = Dispatchers.IO + CoroutineName("Test")
val result1 = context + EmptyCoroutineContext  // 等于 context
val result2 = EmptyCoroutineContext + context  // 等于 context
```

**规则 3：组合是通过链表实现的**
上下文组合实际上形成了一个链表结构，每个节点包含自己的元素并指向下一个上下文。

### 源码分析

```Kotlin
public operator fun plus(context: CoroutineContext): CoroutineContext =
    if (context === EmptyCoroutineContext) this else // fast path -- avoid lambda creation
        context.fold(this) { acc, element ->
            val removed = acc.minusKey(element.key)
            if (removed === EmptyCoroutineContext) element else {
                // make sure interceptor is always last in the context (and thus is fast to get when present)
                val interceptor = removed[ContinuationInterceptor]
                if (interceptor == null) CombinedContext(removed, element) else {
                    val left = removed.minusKey(ContinuationInterceptor)
                    if (left === EmptyCoroutineContext) CombinedContext(element, interceptor) else
                        CombinedContext(CombinedContext(left, element), interceptor)
                }
            }
        }
```

这么看源码就比较简单了，
第一步：快速路径优化：如果合并的上下文是空的，直接返回当前上下文，避免创建 lambda 和执行 fold 操作。
第二步：这里使用了 fold 操作：
初始值：this（当前上下文）
遍历：context 中的每个 element
操作：对于每个元素，从累积值 acc 中移除相同 Key 的元素，然后处理合并
第三步：强制将拦截器放在尾部，因为在协程执行过程中，获取调度器（ContinuationInterceptor）是一个非常频繁的操作。每次协程恢复执行时都需要检查当前的调度器,现在通过强制让拦截器位于链尾，使得查找可以在 O(1) 时间内完成。

这样我们就了解了`CoroutineContext.plus`这个 Kotlin 协程上下文系统的核心函数，理解这个方法的工作原理对于编写高效、可维护的协程代码至关重要，特别是在需要精细控制协程执行环境的场景中。

---

以上
