---
title: kotlin协程-基础设施篇-协程创建与启动：SafeContinuation
tags: [Kotlin, Android]
date: 2025-11-24 11:06:38
toc: true
keywords: 协程基本概念,协程的构造,函数的挂起,协程的上下文,协程的拦截器
---

种一颗树的最好时机是十年前，其次是现在。
学习也一样。
跟着霍老师的《深入理解 Kotlin 携程》学习一下协程。

在这里，我们将 kotlin 中的协程实现分为两个层次

- 基础设施层：标准的协程 API，主要对协程提供了概念和语义上最基本的支持。
- 业务框架层：协程的上层框架支持

## 协程的构造

我们可以很快捷的创建一个简单的协程

```kotlin
    val continuation = suspend {
        println("In Coroutine")
        5
    }.createCoroutine(object : Continuation<Int>{

        override fun resumeWith(result: Result<Int>) {
            println("Coroutine end :$result")
        }
        override val context: CoroutineContext = EmptyCoroutineContext
    })
```

### 函数解析

接下来我们仔细的探查一下这个`createCoroutine`函数。
首先是它的声明：

```kotlin
public fun <T> (suspend () -> T).createCoroutine(
    completion: Continuation<T>
): Continuation<Unit> =
```

这里的`(suspend () -> T)`是函数`createCoroutine`的 Receiver(接收者)，简单解释一下：

#### 带接收者的函数类型

是一种特殊的函数类型，它允许您在函数类型中指定一个接收者对象，使得在函数体内可以直接访问该接收者对象的成员函数和属性。这种函数类型的语法是在函数类型声明之前添加接收者类型。
带接收者的函数类型的语法如下：

> 接收者类型.() -> 返回类型

#### 举个例子：

```kotlin
data class Person(val name: String)

// 带接收者的函数类型
val greeting: Person.() -> String = {
    "Hello, $name!"
}

// 扩展函数
fun Person.greet(): String {
    return "Hello, $name!"
}
fun main() {
    val person: Person = Person("huang")

    // 使用带接收者的函数类型调用函数
    val message1 = person.greeting()

    // 使用扩展函数调用函数
    val message2 = person.greet()

    println(message1) // 输出: Hello, huang!
    println(message2) // 输出: Hello, huang!
}
```

详细了解可以看[kotlin 专栏](https://blog.csdn.net/huangyuan_xuan/category_12990431.html)中的这一篇，
[Kotlin 中的函数类型及 Lambda 表达式：SAM 转换,带接收者的函数类型,匿名函数](https://blog.csdn.net/huangyuan_xuan/article/details/148993630).

#### createCoroutine 说明

在了解了上面的知识之后，我们再回过头来看下`createCoroutine`函数

- Receiver 是一个被 suspend 修饰的`挂起函数`,这也是协程要执行的代码，就先叫它`协程体`
- 它需要一个`Continuation<T>`类型的参数`completion`,这个参数会在协程执行完后调用，实际上就是协程的完成回调
- 返回值是一个` Continuation<Unit>`对象，由于`createCoroutine`只是创建了协程，后面我们还需要使用这个返回对象`启动`协程

## 协程启动

启动协程非常简单，只需要调用`continuation.resume(Unit)`就可以了。

### continuation 解释

下面我们简单看一下`continuation`这个对象是啥。打上断点，我们可以看到它是一个`SafeContinuation`实例。
![continuation](image/kotlin/safe_continuation.png)
我们再回过头看下`createCoroutine`这个函数，实际上返回的就是`SafeContinuation`实例
![SafeContinuation](image/kotlin/create_coroutine.png)
这里传入了两个参数，一个是`delegate`和一个`COROUTINE_SUSPENDED`。
当我们执行 resume 的时候，实际上调用的是`delegate`的`resumeWith`方法。
![resume](image/kotlin/delegate_resume_with.png)
我们进入`SafeContinuation`类中，发现该类的`resumeWith`实际上调用的就是上面传入的`delegate`的`resumeWith`方法,并且这里的 delegate 类型是`HelloKt$main$continuation$1`这个类型特别像是匿名内部类的名字，仔细观察会发现这个匿名类的名字是`<FileName>Kt$<FunctionName>$continuation$1`这种形式，猜测应该是编译器在编译的过程中添加了一些现在我们还暂时用不到的东西，让我们传入的 suspend Lambda 表达式生成了匿名内部类

![resumeWith](image/kotlin/delegate_class_name.png)
当执行`delegate.resume`方法时，我们的`suspend Lambda 表达式`得到执行，并且在执行完毕后，调用了传入的`completion`参数的 resumeWith 方法。

### 其他方式

其实还有一个更方便的创建并启动协程的方法

```kotlin
    suspend {
        println("In Coroutine")
        5
    }.startCoroutine(object : Continuation<Int>{

        override fun resumeWith(result: Result<Int>) {
            println("Coroutine end :$result")
        }
        override val context: CoroutineContext = EmptyCoroutineContext
    })
```

![startCoroutine](image/kotlin/start_coroutine.png)
实际上只是帮我们调用了一下 resume， 哈哈哈，当然了，返回值也不一样了。

### 协程体的 Receiver

除了上面提到的两个方法外，我们还注意到另外两个方法
![createCoroutine receiver](image/kotlin/create_coroutine_reveiver.png)
![startCoroutine receiver](image/kotlin/start_coroutine_receiver.png)
这两个方法和上面的两个方法，只是多了一个协程体的 Receiver 类型 R。这个 R 可以为协程体提供一个作用域，在协程体内可以直接使用作用域内提供的函数或者字段。

#### 怎么用

假设我们有一个数据类

```kotlin
data class Student(val name: String, val age: Int)
```

创建协程并启动

```kotlin
 val coroutineBlock : suspend Student.() -> Int = {
     println("In Coroutine ${this.name}")
     this.age
 }

val continuation = coroutineBlock.createCoroutine(Student("John", 23), completion = object : Continuation<Int> {
     override val context: CoroutineContext
         get() = EmptyCoroutineContext
     override fun resumeWith(result: Result<Int>) {
         println("Coroutine end :$result")
     }

 })
continuation.resume(Unit)
```

这样，我们就可以在`suspend Lambda表达式`中使用 Student 类提供函数和字段了。

---

以上就是协程的创建和启动以及相关的一些零碎知识点，接下来我们一起学习一下**函数的挂起**
