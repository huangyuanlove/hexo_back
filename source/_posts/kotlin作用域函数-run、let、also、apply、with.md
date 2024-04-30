---
title: kotlin作用域函数:run、let、also、apply、with
tags: [Kotlin,Android]
date: 2024-04-28 16:23:11
keywords: 作用域函数,run,let,also,apply,with,作用域函数如何选择
---
刚开始学习 kotlin 的时候，对于这些作用域函数一头雾水，搞不明白为什么要弄出来这么多东西。现在来看看他们具体的区别以及适用的场景。
Kotlin 标准库包含几个函数，它们的唯一目的是在对象的上下文中执行代码块。 当对一个对象调用这样的函数并提供一个`lambda表达式`时，它会形成一个临时作用域。在此作用域中，可以访问该对象而无需其名称。这些函数称为作用域函数。 共有以下五种：`let`、`run`、`with`、`apply`以及`also`。
废话不多说，先把从 kotlin 官方上扒拉下来的结论放这里
<!--more-->
[作用域函数中文版](https://book.kotlincn.net/text/scope-functions.html)
[作用域函数英文版](https://kotlinlang.org/docs/scope-functions.html)

### 总结在前面
文章太长太啰嗦，直接看这里的结论：

|函数	|对象引用	|返回值	|是否是扩展函数|
|--|--|--|--|
|let|	it|	Lambda表达式结果	|是|
|run|	this	|Lambda表达式结果	|是|
|run|	-	|Lambda表达式结果	|不是：调用无需上下文对象|
|with|	this	|Lambda表达式结果	|不是：把上下文对象当做参数|
|apply|	this	|上下文对象|	是|
|also|	it|	上下文对象|	是|

以下是根据预期目的选择作用域函数的简短指南：

* 对一个非空（non-null）对象执行 lambda 表达式：let
* 将表达式作为变量引入为局部作用域中：let
* 对象配置：apply
* 对象配置并且计算结果：run
* 在需要表达式的地方运行语句：非扩展的 run
* 附加效果：also
* 一个对象的一组函数调用：with
不同作用域函数的使用场景存在重叠，可以根据项目或团队中使用的特定约定来选择使用哪些函数。

<font color='red'>虽然作用域函数可以让代码更加简洁，但是要避免过度使用它们：这会使代码难以阅读并可能导致错误。 我们还建议避免嵌套作用域函数，同时链式调用它们时要小心：因为很容易混淆当前上下文对象与`this`或`it`的值。</font>

### 使用示例

假如我们有这么一个数据类
``` kotlin
data class Book(var name: String, var price: Int) {
    fun changePrice(price: Int) {
        this.price = price
    }
}
val book = Book("book name", 68)
```

#### 函数声明
``` kotlin

public inline fun <T> T.also(block: (T) -> Unit): T
public inline fun <T> T.apply(block: T.() -> Unit): T 

public inline fun <T, R> T.let(block: (T) -> R): R
public inline fun <T, R> T.run(block: T.() -> R): R

public inline fun <R> run(block: () -> R): R

public inline fun <T, R> with(receiver: T, block: T.() -> R): R
```
我们把看起来相近的作用域函数的声明放在一块对比着看，看到这里就清楚了的就不要往下看了，看了也是浪费时间。


#### also
函数声明
``` kotlin
public inline fun <T> T.also(block: (T) -> Unit): T
```
`also`函数是对泛型 T 的扩展函数，接收一个参数类型为T、无返回值(返回值为Unit类型)的函数，且`also`函数的返回值就是调用者。
* 上下文对象作为 lambda 表达式的参数（it）来访问。
* 返回值是上下文对象本身。

对于执行一些将上下文对象作为参数的操作很有用。 对于需要引用对象而不是其属性与函数的操作，或者不想屏蔽来自外部作用域的 this 引用时，请使用 also。
当你在代码中看到 also 时，可以将其理解为**并且用该对象执行以下操作**
``` kotlin
val alsoResult = book.also {
    it.changePrice(20)
    it.name = "alsoResult"
}
println("alsoResult $alsoResult")
```
这里打印结果是`alsoResult Book(name=alsoResult, price=20)`,看源码的话，可以简单的里面为调用了一下传入的函数，然后返回了调用者
``` kotlin
public inline fun <T> T.also(block: (T) -> Unit): T {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    block(this)
    return this
}
```

#### apply
函数声明
``` kotlin
public inline fun <T> T.apply(block: T.() -> Unit): T
```
可以看得出来`apply`是泛型 T 的扩展函数，接收一个带有 T 类型接收者的无参、无返回值的函数，并且`apply`函数返回值就是 T 类型，也就是调用者的类型。因为这里参数中的 T 是作为接收者类型，而不是参数，所以在传入的函数中需要用`this`而非`it`来指代调用者。
用法和`also`相差无几，只不过一个是接收者类型，一个是参数。

* 上下文对象 作为接收者（this）来访问。
* 返回值 是上下文对象本身。

对于不返回值且主要在接收者（this）对象的成员上运行的代码块使用它。apply最常见的使用场景是用于对象配置。这样的调用可以理解为**将以下赋值操作应用于对象**。

``` kotlin
val applyResult = book.apply {
    changePrice(200)
    name = "applyResult"
}
println("applyResult $applyResult")
```
这里打印的结果是`applyResult Book(name=applyResult, price=200)`.
源码也和`also`几乎一样
``` kotlin
public inline fun <T> T.also(block: (T) -> Unit): T {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    block(this)
    return this
}
```


#### let
函数类型声明如下：
``` kotlin
public inline fun <T, R> T.let(block: (T) -> R): R
```
可以看到，let 是对泛型 T 的扩展函数，该扩展函数接收一个函数参数，并且函数参数的接收一个 T 类型的参数，且返回值是 R 类型，也是`let`这个扩展函数的返回值类型。
* 上下文对象作为 lambda 表达式的参数（it）来访问。
* 返回值是 lambda 表达式的结果。
  
``` kotlin
val letResult = book.let {
    it.changePrice(100)
    it.name = "letResult"
}
println("letResult $letResult")
```
这里传入的是一个 Lambda 表达式，前面说过，对于单参数值的Lambda 表达式，参数会被隐式声明为`it`,当然我们也可以指定一个具名意义的变量，比如
``` kotlin
val letResult = book.let { bookEntry: Book ->
    bookEntry.changePrice(100)
    bookEntry.name = "letResult"
}
```
这里打印的结果是`letResult kotlin.Unit`。因为对于 Lambda 表达式来讲，如果最后一条语句是非赋值语句，则返回该语句的值；如果是赋值语句，则返回 Unit。
我们可以这么写来返回我们需要的值：
``` kotlin
val letResult = book.let {
    it//返回值就是传入的 book 对象
}
val letResult = book.let {
    1//返回值就是1
}
val letResult = book.let {
     return@let 1//之前的文章中说过的显示指定返回值，是 1
}
```
从另外一个角度看，`let`和 `also`、`apply`也差不多，只不过多了一个返回值类型，返回值就是传入的 Lambda 表达式的返回值
源码也差不了多少
``` kotlin
public inline fun <T, R> T.let(block: (T) -> R): R {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    return block(this)
}
```

* let 可用于在调用链的结果上调用一个或多个函数。
* let 经常用于执行包含非空值代码块。如需对非空对象执行操作， 可对其使用安全调用操作符`?.`并调用 let 在 lambda 表达式中执行操作。

#### run
`run`这个函数给了两种方式
``` kotlin
public inline fun <T, R> T.run(block: T.() -> R): R
public inline fun <R> run(block: () -> R): R
```
先看第一种，看起来就是把`let`中函数参数中的 T 类型参数改成了接收者类型，也是返回 R 类型；这和`apply`与`also`的区别是一样的。
* 上下文对象 作为接收者（this）来访问。
* 返回值 是 lambda 表达式结果。

当 lambda 表达式同时初始化对象并计算返回值时，run 很有用。
``` kotlin
val runResult = book.run {
    name = "runResult"
    changePrice(110)
    this //作为返回值
}
println("runResult $runResult")
```
源码是这样的
``` kotlin
public inline fun <T, R> T.run(block: T.() -> R): R {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    return block()
}
```
第二种
``` kotlin
val otherRunResult =  run {
    Book("run", 120) //作为返回值
}
println("otherRunResult $otherRunResult")
```
源码
``` kotlin
public inline fun <R> run(block: () -> R): R {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    return block()
}
```
这也没啥好说的，只不过是这里并没有输入参数，只是可以使你在需要表达式的地方就可以执行一个语句。

#### with
函数声明
``` kotlin
public inline fun <T, R> with(receiver: T, block: T.() -> R): R
```
`with`并不是扩展函数，需要传入一个T 类型的receiver，可以在 block 中访问这个receiver的方法和属性，
* 上下文对象作为接收者（this）使用。
* 返回值是 lambda 表达式结果。

建议当不需要使用 lambda 表达式结果时，使用 with 来调用上下文对象上的函数。 在代码中，with 可以理解为**对于这个对象，执行以下操作.**
``` kotlin
val withResult = with(book) {
    changePrice(300)
    name = "withResult"
    this //作为返回值
}
println("withResult $withResult")
```
这里的打印结果是`withResult Book(name=withResult, price=300)`

### 如何选择
这里再搬运一个总结的表格
<table>
    <tr> 
        <th >函数名</th> 
        <th >作用</th> 
        <th >应用场景</th> 
        <th >备注</th> 
    </tr>
    <tr>
      <td>let</td>
      <td rowspan="2">定义一个变量在特定作用域内<br/>统一做判空处理</td>
      <td rowspan="2">明确一个变量所处特定的作用域范围内可使用<br/>针对一个可空对象统一做判空处理</td>
      <td rowspan="2">区别在于返回值<br/>let函数：返回值=最后一行|return的表达式<br/>also函数：返回值=传入对象本身</td>
    </tr>
    <tr>
      <td>also</td>
    </tr>
    <tr>
      <td>with</td>
      <td>调用同一个对象的多个方法|属性时，可以省去对象名，直接调用方法、访问属性</td>
      <td>需要多次调用同一个对象的属性|方法</td>
      <td>返回值=最后一行|return表达式</td>
    </tr>
    <tr>
      <td>run</td>
      <td rowspan="2">结合了let 函数和 with 函数的作用</td>
      <td>1.调用同一个对象的多个方法/属性时可以省去对象名重复，直接调用方法名 /属性即可<br/>2.定义一个变量在特定作用域内<br/>3.统一做判空处</</td>
      <td>优点:避免了let函数必须使用it参数替代对象弥补了with函数无法判空的缺点</td>
    </tr>
    <tr>
      <td>apply</td>
      <td>对象实例初始化时需要对对象中的属性进行赋值且返回该对象</td>
      <td>二者区别在于返回值:<br/>run函数返回最后一行的值|表达式<br/>apply函数返回传入的对象的本身</td>
    <tr>
</table>

### 另外一个角度的选择

#### it or this
每个作用域函数都使用以下两种方式之一来引用上下文对象
1. 作为 lambda 表达式的接收者 （this）
2. 作为 lambda 表达式的参数（it）
  
两者都提供了同样的功能，`run`、`with`以及`apply`通过关键字`this`将上下文对象引用为`lambda`表达式的接收者。 因此，在它们的`lambda表达式`中可以像在普通的类函数中一样访问上下文对象。在大多数场景，当你访问接收者对象时你可以省略`this`， 来让你的代码更简短。 相对地，如果省略了`this`，就很难区分接收者对象的成员及外部对象或函数。因此，对于主要对对象的成员进行操作（调用其函数或赋值其属性）的lambda表达式， 建议将上下文对象作为接收者（this）。
反过来，`let`及`also`将上下文对象引用为`lambda表达式参数`。如果没有指定参数名，对象可以用隐式默认名称`it`访问。`it`比`this`简短，带有`it`的表达式通常更易读。不过，当调用对象函数或属性时，不能像`this`这样隐式地访问对象。 因此，当上下文对象在作用域中主要用作函数调用中的参数时，通过`it`访问上下文对象会更好。 在代码块中使用多个变量时，`it`也更好一些。

#### 返回值
根据返回结果，作用域函数可以分为以下两类：

apply 及 also 返回上下文对象。
let、run 及 with 返回 lambda 表达式结果.
apply 及 also 的返回值是上下文对象本身。因此，它们可以作为辅助步骤包含在调用链中：可以继续在同一个对象上一个接一个地进行链式函数调用。

### 写在最后的注意事项

在最开始的红色部分也提高过尽量不要嵌套使用作用域函数，警惕引发的上下文混淆。看下面的代码猜一下打印结果是什么。
``` kotlin
fun main() {
    val length = 0
    "hello".apply {
        println("this is apply $length")
        println("this is apply ${this.length}")
    }

    "hello".let {
        println("this is let $it")
        "world".also {
            println("this is run $it")
        }
    }

    fun innerFunc(){
        "hi".apply {
            println("this is innerFunc apply $length")
            println("this is innerFunc apply ${this.length}")

        }
    }
    innerFunc()
}
```
结果是如下：
>this is apply 0
>this is apply 5
>this is let hello
>this is run world
>this is innerFunc apply 0
>this is innerFunc apply 2

这里我们在写代码的时候，IDE 给了提示:**Implicit parameter 'it' of enclosing lambda is shadowed**
![Implicit parameter 'it' of enclosing lambda is shadowed ](image/kotlin/scope_func_implicit_param.png)
我们可以通过修改隐式 it 的名字来避免这个问题
``` kotlin
"hello".let {
    println("this is let $it")
    "world".also { world->
        println("this is run $world")
    }
}
```
但最好还是避免这种嵌套调用的情况




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

* 关键字
  * <input type='checkbox' disabled='true' checked>作用域函数：with、let、run、apply、also</input>

未学习：
* 关键字
  * object
  * Unit
  * Nothing
  * inline,noinline,crossinline


* 泛型
   * 逆变
   * 协变
* 委托
   * 委托类
   * 委托属性
   * 自定义委托

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
  
