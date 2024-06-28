---
title: kotlin中的内联函数
tags: [Kotlin,Android]
date: 2024-06-26 15:06:32
keywords: kotlin,内联函数,inline,noinline,crossinline,参数内联,非局部返回
---
众所周知，在 kotlin 中函数是一等公民，在源码、各种框架中都能看到高阶函数的身影，我们也发现伴随着高阶函数的还有几个关键字：`inline`,`noinline`,`crossinline`。那这些关键字有什么作用？应该如何使用？
<!--more-->

### inline
`inline`关键字用于指示编译器将函数及其参数内联展开到调用处。内联函数可以减少函数调用的开销，并允许非局部返回
**作用**：
* 减少函数调用开销：通过内联展开，消除了函数调用的开销。
* 允许非局部返回：内联函数的 lambda 参数可以使用 return 从外部函数返回。

先看一下没有`inline`修饰的情况
``` kotlin
inline  fun hello(){
    println("hello")
}

fun sayHi(){
    println("hi")
}

fun main() {
    hello()
    sayHi()
}
```
再看一下反编译成 java 代码的样子
``` Java
public final class InlineKt {
   public static final void hello() {
      int $i$f$hello = false;
      String var1 = "hello";
      System.out.println(var1);
   }

   public static final void sayHi() {
      String var0 = "hi";
      System.out.println(var0);
   }

   public static final void main() {
      int $i$f$hello = false;
      String var1 = "hello";
      System.out.println(var1);
      sayHi();
   }

   // $FF: synthetic method
   public static void main(String[] args) {
      main();
   }
}
```
可以看到，被`inline`修饰的代码直接展开复制到了调用的地方，好处是什么？少了一层调用栈，减少了开销。坏处：函数体被展开复制到了调用的地方，编译后的产物体积肯定会增大。
那这样的话，为啥也要有`inline`关键字嘞，看着也没啥用。其实除了可以**内联自己内部的代码**，还可以**内联作为参数的方法代码**
``` kotlin
inline fun hello(postAction:()->Unit){
    println("hello")
    postAction()
}
fun sayHi(postAction:()->Unit){
    println("hi")
    postAction()
}

fun main(){
    hello { println("hello lambda") }
    sayHi { println("sayHi lambda") }

    hello (fun(){
        println("hello")
    })
    sayHi (fun(){
        println("sayHi")
    })
}
```
众所众知，Java 中是不支持函数作为参数传递的，但 kotlin 可以，那么转成字节码运行在 jvm 上是怎么处理的？办法是将其包装成一个对象来调用。
看反编译成 java 的代码
``` Java
public static final void main() {
  int $i$f$hello = false;
  String var1 = "hello";
  System.out.println(var1);
  int var2 = false;
  String var3 = "hello lambda";
  System.out.println(var3);
  sayHi((Function0)null.INSTANCE);
  $i$f$hello = false;
  var1 = "hello";
  System.out.println(var1);
  var2 = false;
  var3 = "hello";
  System.out.println(var3);
  sayHi((Function0)null.INSTANCE);
}
```
可以看到，在调用`sayHi`的地方实际是创建了一个`Function0`对象进去，也许创建者一次对象的开销可以湖绿，但如果是用在频繁调用的场景下呢？比如页面刷新绘制、循环等等等等。如果真的是这样，这不就有可能会造成面试中经常问到的`内存抖动`么。
所以，这种时候，我们使用`inline`可以减少参数对象的创建，从而避免出现一些问题。
但是，我们也不能看见频繁调用的函数就加上`inline`，毕竟谁也不会为了减少一次调用栈，把函数体直接复制到每个调用的地方吧？主要还是用在高阶函数上，并且根据函数调用的情况综合来判断是否可以使用`inline`。

### noinline

noinline 关键字用于标记不应该内联的 lambda 参数。默认情况下，内联函数的所有 lambda 参数都会被内联展开，但有时我们可能希望某些 lambda 参数不被内联。

**作用**：
* 防止内联：阻止特定的 lambda 参数被内联展开。
* 保留 lambda 参数：适用于需要将 lambda 参数作为对象传递的情况。


既然`inline`是一种优化，假设使用者也经过考虑，将函数用`inline`修饰，那为什么还会有`noinline`这个关键字？
先来思考一个问题：kotlin 中一切都是对象，函数也能作为参数或者返回值，那被内联的函数参数作为参数或者返回值时会怎么样？
答案是不可以，因为被内联的函数已经被展开了，不再是一个对象了，那怎么办？加上`noinline`，告诉编译器，这个函数参数不要进行内联。
这里也有一个例外情况，被内联的函数参数，可以作为其他内联函数的参数。为啥？因为被内联函数被展开复制到调用处了哇。
看个例子：
``` kotlin
inline fun hello(preAction:()->Unit, postAction:()->Unit):()->Unit{
    preAction()
    println("hello")
    postAction()
    another(postAction)
    /**
     * Illegal usage of inline-parameter 'postAction' in 'public inline fun hello(preAction: () -> Unit, postAction: () -> Unit): () -> Unit defined in root package in file Inline.kt'. Add 'noinline' modifier to the parameter declaration
     */
    anotherInline(postAction)
    return postAction
    /**
     * Illegal usage of inline-parameter 'postAction' in 'public inline fun hello(preAction: () -> Unit, postAction: () -> Unit): () -> Unit defined in root package in file Inline.kt'. Add 'noinline' modifier to the parameter declaration
     */
}

fun another(action:()->Unit){
    action()
}
inline fun anotherInline(action:()->Unit){
    action()
}
```
这里调用`another(postAction)` 和 `return postAction`时，IDE 会报错，提示需要加上`noinline`。
也就是说，如果 inline 函数参数中有函数对象，并且这个函数对象需还需要充当其他非 inline 函数的参数或者充当返回值，那么就需要加上`noinline`,还有个偷懒的办法，IDE告诉你需要加，那就加上。


### crossinline

crossinline 关键字用于标记 lambda 参数，保证它们不会进行非局部返回。crossinline 参数不能使用 return 从外部函数返回。
**作用**：
 防止非局部返回：确保 lambda 参数不会从外部函数返回。
 安全性：在某些情况下，防止非局部返回可以避免编译错误或逻辑问题。

这里有个词是`非局部返回`,什么意思呢？先看个例子
``` kotlin
inline fun hello( postAction: () -> String) {
    println("hello")
    postAction()
}
fun main() {
    hello {
        println("second hi")
        return //猜这里是哪个函数的返回
    }
    println("after second hi\n")
}
```
会发现`after second hi`没有打印，结束的是`main`函数而不是`hello`函数，但这里就会有个歧义，`return`结束哪个函数，需要看调用者是不是`inline`,这就挺郁闷的，所以这里就有了一个规定：
> lambda表达式中不允许直接 return，除非是当做内联函数的参数。
> 不能直接 return，但允许使用 return@label方式进行返回，结束 label 处的函数,这里的 label值可以自定义，但一般默认是调用的函数名字

所以当我们这么写的时候
``` kotlin
fun hi( postAction: ()->Unit){
    println("hi")
    postAction()
}
fun main() {
    hi {
        println("hi")
        return//错误，提示 'return' is not allowed here
    }
}
```
在 return 处会提示`'return' is not allowed here`,但如果我们一定要写，可以写成
``` kotlin
hi {
    println("hi")
    return@hi
}
```
到这里还没有`crossinline`的什么事，但想一想，如果多套一层：传入的函数参数，又作为其他函数的参数调用呢？比如这样
``` kotlin
inline fun hello(postAction: () -> Unit) {
    println("hello")
    doAction { postAction() }//注意这里
    run { postAction() }

}

fun doAction(postAction: () -> Unit) {
    postAction()
}
```

注意上面 doAction 的调用，是不允许这样写的，会给出报错提示：
> Can't inline 'postAction' here: it may contain non-local returns. Add 'crossinline' modifier to parameter declaration 'postAction'

意思是这种`间接调用`可能会导致非本地返回问题，也就是说我不知道你传入的函数参数中有没有 return，如果有的话，又会造成上面说的那个问题。那怎么办？在postAction参数前面加上`crossinline`修饰符，这样就可以间接调用了。不过这又带来了一个新问题
``` kotlin
inline fun hello(crossinline postAction: () -> Unit) {
    println("hello")
    doAction { postAction() }
    run { postAction() }

}

fun doAction(postAction: () -> Unit) {
    postAction()
}
fun main() {
  hello {
    println("hi")
    return //错误 提示：'return' is not allowed here
  }
}
```
会发现传入的Lambda 表达式中不允许这种直接 return 了，但还是可以使用 return@label 进行返回的。
但是你说：我既要又要怎么办？
抱歉，没办法，自己玩吧.

----
参考
[内联函数](https://book.kotlincn.net/text/inline-functions.html)建议把函数这一节都看一下  
[Inline functions](https://kotlinlang.org/docs/inline-functions.html)
[Kotlin 源码里成吨的 noinline 和 crossinline 是干嘛的？看完这个视频你转头也写了一吨](https://rengwuxian.com/kotlin-source-noinline-crossinline/)  



---- 

已学习：
* 扩展
  * <input type='checkbox' disabled='true' checked>扩展函数</input>
  * <input type='checkbox' disabled='true' checked>扩展属性</input>
  * <input type='checkbox' disabled='true' checked>作用域</input>

* 函数类型
  * <input type='checkbox' disabled='true' checked>带有接收者的函数类型</input>
  * <input type='checkbox' disabled='true' checked>Lambda表达式</input>
  * <input type='checkbox' disabled='true' checked>SAM 转换</input>
* 泛型
  * <input type='checkbox' disabled='true' checked>逆变</input>
  * <input type='checkbox' disabled='true' checked>协变</input>
  * <input type='checkbox' disabled='true' checked>类型投影</input>
  * <input type='checkbox' disabled='true' checked>星投影</input>
  * <input type='checkbox' disabled='true' checked>泛型约束</input>
* 关键字
  * <input type='checkbox' disabled='true' checked>作用域函数：with、let、run、apply、also</input>
  * <input type='checkbox' disabled='true' checked>object:匿名内部类、单例模式、伴生对象</input>
  * <input type='checkbox' disabled='true' checked>Unit、Nothing</input>
  * <input type='checkbox' disabled='true' checked>inline,noinline,crossinline</input>
* 委托
  * <input type='checkbox' disabled='true' checked>委托类</input>
  * <input type='checkbox' disabled='true' checked>委托属性</input>
  * <input type='checkbox' disabled='true' checked>自定义委托</input>

未学习：

* 协程
    * 启动
    * 挂起
    * Job
    * Context
    * Channel
    * Flow
    * select
    * 并发、异常
    * launch
    * Dispatchers
    * CoroutineScope