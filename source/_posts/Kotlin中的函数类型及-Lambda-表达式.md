---
title: Kotlin中的函数类型及 Lambda 表达式
tags: [Kotlin,Android]
date: 2024-04-23 11:41:47
keywords: Kotlin,函数类型,高阶函数,Lambda表达式,SAM转换,带接收者的函数类型,匿名函数
---
继续上次在扩展函数中遗留下的问题：函数类型。总所周知，在Kotlin 中函数是一等公民。那么什么是高阶函数嘞？到处翻了翻，发现差不多是这么个意思：**接受一个或多个函数作为参数，或者返回一个函数。**在 Kotlin 协程中的 launch、async函数以及各种各样的框架中到处都是高阶函数的影子，称高阶函数是 Kotlin函数式编程、各大框架的基石也不过分。
<!-- more -->

不得不说，这里面概念挺多的，有的时候我们会用，但并不知道叫什么名字。有的知道名字但不知道是什么东西。
* 高阶函数
* 扩展函数
* Lambda
* SAM转换
* 带接收者的函数类型
* 带有接收者的函数字面值
  
问题不大，慢慢整，慢慢理解，多读几遍文档，参考一下别人的看法，也就熟悉了。


### 函数类型
这里想表达的并不是说kotlin 中的函数分类，比如什么内联函数、扩展函数、标准函数、高阶函数等等这种分类，而是说在函数作为返回值或者参数的时候，我们怎么确认这个函数就是我们想要的类型，或者说如何使用编程语言来描述一个函数。比如我们在调用函数的时候传入的参数，我们会讲这个函数需要一个 Int 类型的参数，那如果我们调用的函数需要另外一个函数作为参数我们应该怎么表示嘞？这里就引出了函数类型。
举个例子
``` kotlin
fun functionA():Unit{
    println("functionA")
}
fun functionA1(name:String):Unit{
    
}
fun functionA11(name:String):String{
    return "hi $name"

}
```
我们应该如何描述上面的三个方法嘞？
> functionA,不需要参数，返回值为 Unit
> functionA1, 需要一个String 类型的参数，返回值为 Unit
> functionA11,需要一个String 类型的参数，返回值为 String

那么在 kotlin 编程语言中又是如何描述的？
![](image/kotlin/kotlin_function0<Unit>.png)
![](image/kotlin/kotlin_function1<String,Unit>.png)
![](image/kotlin/kotlin_function1<String,String>.png)

上面的图是将鼠标悬停在变量上就会出现，当然也可以选中变量或者表达式，按 ctrl+shift+p来显示类型

可以看到在`kotlin`中是用`KFunction0<Unit>`、`KFunction1<String, Unit>`、`KFunction1<String, String>`这种形式来描述一个函数。这里的 KFunction 后面的数字表示这个函数的参数个数，尖括号中的类型表示参数的类型，最后一个类型表示函数的返回值类型。比如`KFunction1<String, Unit>`表示这个函数需要`1`个`String`类型的参数，返回值类型为`Unit`。而`KFunction1<String, String>`表示这个函数需要`1`个`String`类型的参数，返回值为`String`。
如果函数是挂起函数(被suspend修饰)，则对应的类型为` KSuspendFunction0<Unit>`,以此类推。
那么如果是高阶函数嘞？
```
fun functionC(method:()->String):String{
    return method()
}
suspend fun  suspendFunctionC(method: () -> String): String {
    return method()
}
suspend fun  suspendFunctionC1(method:suspend () -> String): String {
    return method()
}
```
同样的方法，我们可以看到
>`functionC`对应的描述是`KFunction1<() -> String, String>`
>`suspendFunctionC`对应的描述是` KSuspendFunction1<() -> String, String>`
>`suspendFunctionC1`对应的描述是` KSuspendFunction1<suspend () -> String, String>`。

对于扩展函数也一样
``` kotlin
fun String.A1() {
    println(this)
}

fun String.A11(): String {
    return this
}    
val stringA1 = String::A1 // KFunction1<String, Unit>
val stringA11 = String::A11 // KFunction1<String, String>
```

**注意**，我们还可以使用`typealias`给函数类型取一个别名
`typealias ClickHandler = (Button, ClickEvent) -> Unit`



### 带接收者的函数类型
一种特殊的函数类型，它允许您在函数类型中指定一个接收者对象，使得在函数体内可以直接访问该接收者对象的成员函数和属性。这种函数类型的语法是在函数类型声明之前添加接收者类型。
带接收者的函数类型的语法如下：
> 接收者类型.() -> 返回类型

通过使用带接收者的函数类型，我们可以创建具有接收者的函数变量、函数参数或函数返回类型，以便在调用函数时可以直接操作接收者对象。这样可以实现一种类似扩展函数的效果。举个例子：
``` kotlin
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
总结起来，带接收者的函数类型更适合在函数类型的声明和传递中使用，以提供特定上下文的函数操作。而扩展函数则更适合在已有类上添加新的函数，使得在调用该类时可以使用这些额外的函数。


### 小总结
先小小的总结一下：
* 所有函数类型都有一个圆括号括起来的参数类型列表以及一个返回类型：(A, B) -> C 表示接受类型分别为 A 与 B 两个参数并返回一个 C 类型值的函数类型。 参数类型列表可以为空，如 () -> A。Unit 返回类型不可省略。
* 函数类型可以有一个额外的接收者类型，它在表示法中的点之前指定： 类型 A.(B) -> C 表示可以在 A 的接收者对象上以一个 B 类型参数来调用并返回一个 C 类型值的函数。 `带有接收者的函数字面值`通常与这些类型一起使用。
* 挂起函数属于函数类型的特殊种类，它的表示法中有一个 suspend 修饰符 ，例如 suspend () -> Unit 或者 suspend A.(B) -> C。
* 如需将函数类型指定为可空，请使用圆括号，如下所示： ((Int, Int) -> Int)?。


这样我们在看其他框架的时候就知道框架中的高阶函数怎么调用了：
> 当参数类型为`() -> String`时，我们需要传入一个没有参数且返回值为String类型的函数，对应的类型是`KFunction0<String>`

### 函数实例化

既然函数也是对象，那么理所当然的可以被实例化。我们可以使用以下几种方式获取函数类型的实例
* 使用函数字面值的代码块
  * lambda 表达式: { a, b -> a + b },
  * 匿名函数: fun(s: String): Int { return s.toIntOrNull() ?: 0 }
* 使用已有声明的可调用引用
  * 顶层、局部、成员、扩展函数：::isOdd、 String::toInt，
  * 顶层、成员、扩展属性：List<Int>::size，
  * 构造函数：::Regex
* 使用实现函数类型接口的自定义类的实例：
  ``` kotlin
  class IntTransformer: (Int) -> Int {
    override operator fun invoke(x: Int): Int = TODO()
  }
  val intFunction: (Int) -> Int = IntTransformer()
  ```


### 有无Receiver的函数相互转化
带与不带接收者的函数类型非字面值可以互换，其中接收者可以替代第一个参数，反之亦然。例如，(A, B) -> C 类型的值可以传给或赋值给期待 A.(B) -> C 类型值的地方，反之亦然。这也是为什么`String.A1()`明明没有声明需要参数，为啥和上面的`functionA1`方法是相同的类型嘞？可以这么认为:Kotlin中的扩展函数将接收者本身当做第一个参数传入，要不然为啥在`String.A1()`里面可以使用`this`来代替调用者本身嘞？
那既然这样的话，也就是说这两者是可以互换的。
``` kotlin
fun d(block :(String) -> Unit) {
   block("hello");
}
d(String::A1)
d(::functionA1)
```
需要注意的是，这里仅针对在引用和调用时可以互相转换，比如
``` kotlin
val sayHi: (String) -> Unit = { name:String->  println("hi $name") }
sayHi.invoke("huangyuan")
sayHi("huangyuan")

val sayHello: String.() -> Unit = { println("hello $this") }
sayHello.invoke("huangyuan")
sayHello("huangyuan")
"huangyuan".sayHello()

val sayHiRef:(String)->Unit =sayHi
val sayHiRef1: String.() -> Unit = sayHi
```
但是如果将 sayHello 和sayHi这两个函数等号右边互换一下则会报错。

需要注意的是这里还有一个概念：`带接收者的函数字面值`（Function Literals with Receiver），也称为带接收者的 Lambda 表达式，是一种特殊的 Lambda 表达式。它允许在 Lambda 表达式中访问特定类型的对象的成员，就像在该对象的成员函数中一样。通过使用带接收者的函数字面值，可以在 Lambda 表达式中以更简洁的方式操作特定类型的对象。上面对`sayHello`的定义就属于这种形式。
也就是说：带有接收者的函数类型，例如 A.(B) -> C，可以用特殊形式的函数字面值实例化----带有接收者的函数字面值。
这里解释一下：所谓的字面量，就是不用变量名称直接用相对应的值写出来。比如“hello world”就是一个字符串字面量、12.23是一个 Double 的字面量、4是一个 Int 的字面量。

### 函数类型实例调用
既然能获取到函数类型的实例，那么肯定就可以调用了。
这里调用方式有两种，一种是通过`invoke()`，比如`func.invoke()`,或者直接在引用后面加上括号`func()`:
``` kotlin
val functionOne: Int.() -> Unit = { println("aaFunRefRec $this  ") }
functionOne.invoke(10001)
functionOne(10001)

val functionTwo: Int.(String) -> Unit = { println("other $this  ") }
functionTwo.invoke(1, "other")
functionTwo(1, "other")
```
### Lambda表达式
我们在使用Java语言开发Android 应用的时候可能已经体验过 Lambda 表达式了，最常见的就是给 View 设置点击监听的时候
![replace_with_lambda_tip](image/kotlin/replace_with_lambda_tip.png)
当我们点击了之后，代码就成了这样
``` Java
llShowMoreDialog.setOnClickListener(v -> showToast("点击了"));
```
目前在 java 中只能简化成这样的，kotlin 中还可以进一步简化，后面再说。这里先看看Lambda表达式语法：
``` kotlin
val sum: (Int, Int) -> Int = { x: Int, y: Int -> x + y }
```
* lambda 表达式总是括在花括号中。
* 完整语法形式的参数声明放在花括号内，并有可选的类型标注。
* 函数体跟在一个 -> 之后。
* 如果推断出的该 lambda 的返回类型不是 Unit，那么该 lambda 主体中的最后一个（或可能是单个）表达式会视为返回值。

如果Lambda 表达式的参数可以推断出来，我们可以省略一些类型，比如上面的 sum 函数可以省略为
``` kotlin
val sum1 = { x: Int, y: Int -> x + y }
val sum2: (Int, Int) -> Int = { x, y -> x + y }
```
我们在写 Android 时经常会用到给某个控件设置点击事件，就像上面的例子一样
``` kotlin
view.setOnClickListener(object :View.OnClickListener{
    override fun onClick(view: View?) {
        println("click ${view?.id}")
    }
})
view.setOnClickListener { println("click $it ") }
```
#### SAM转换

那么它是怎么从上面使用匿名内部类变成下面样子的？这里就要提一下`SAM转换`了：SAM是Single Abstract Method的缩写，意思就是只有一个抽象方法的类或者接口。但在Kotlin和Java 8里，SAM代表着只有一个抽象方法的接口。只要是符合SAM要求的接口，编译器就能进行SAM转换，也就是我们可以使用Lambda表达式，来简写接口类的参数。
需要注意的是，Java 8中的SAM有明确的名称，叫做`函数式接口(FunctionalInterface)`。FunctionalInterface的限制如下，缺一不可：
* 必须是接口，抽象类不行；
* 该接口有且仅有一个抽象的方法，抽象方法个数必须是1，默认实现的方法可以有多个。
  
同样的，在kotlin中也有限制：
* 必须是函数接口，也就是声明为`fun interface`
* 只能包含一个抽象方法，并且不能包含默认方法

因此,kotlin 编译器会将该方法自动转化为`fun setOnClickListener(l: ((View!) -> Unit)?)`，我们才得以使用 Lambda表达式来简化代码。可以将代码写成这样
``` kotlin
view.setOnClickListener({view:View?-> println("click ${view?.id}")})
```
这种情况下，由于 kotlin 支持类型推导，所以我们可以将`View?`也省略掉，接着还会触发一个被称之为`单个参数的隐式名称`的东西，原话是这么说的
>If the compiler can parse the signature without any parameters, the parameter does not need to be declared and -> can be omitted. 该参数会隐式声明为 it
因此，我们得到了这样子的代码
``` kotlin
view.setOnClickListener({println("click ${it?.id}")})
```
按照 Kotlin 惯例，如果函数的最后一个参数是函数，那么作为相应参数传入的 lambda 表达式可以放在圆括号之外：
``` kotlin
view.setOnClickListener(){println("click ${it?.id}")}
```
这种语法也称为`拖尾lambda(trailing lambda)`表达式。
如果该 lambda 表达式是调用时唯一的参数，那么圆括号可以完全省略：
``` kotlin
view.setOnClickListener{println("click ${it?.id}")}
```
这就是我们最终得到的代码样子

#### 从lambda表达式中返回一个值
这里有两种方式，一种是隐式返回：如果我们什么都不做，将返回最后一个表达式的值。
另外一种就是使用限定的返回语法从lambda显式返回一个值
``` kotlin
ints.filter {
    val shouldFilter = it > 0
    shouldFilter
}

ints.filter {
    val shouldFilter = it > 0
    return@filter shouldFilter
}
```
这两种方式是等价的。
那么这个标签 **@filter**是怎么来的呢？
在 Kotlin 中任何表达式都可以用标签来标记。 标签的格式为标识符后跟 @ 符号，例如：abc@、fooBar@。 要为一个表达式加标签，我们只要在其前加标签即可.
比如我们在嵌套函数中，标签限定的 return 允许我们从外层函数返回，比如从 Lambda 表达式中返回
``` kotlin
fun foo() {
    listOf(1, 2, 3, 4, 5).forEach {
        if (it == 3) return // 非局部直接返回到 foo() 的调用者
        print(it)
    }
    println("this point is unreachable")
}

fun foo() {
    listOf(1, 2, 3, 4, 5).forEach lit@{
        if (it == 3) return@lit // 局部返回到该 lambda 表达式的调用者——forEach 循环
        print(it)
    }
    print(" done with explicit label")
}
```
通常情况下使用**隐式标签**更方便，因为该标签与接受该 lambda 的函数同名。
``` kotlin
fun foo() {
    listOf(1, 2, 3, 4, 5).forEach {
        if (it == 3) return@forEach // 局部返回到该 lambda 表达式的调用者——forEach 循环
        print(it)
    }
    print(" done with implicit label")
}
```

官方这里也给了一个提示：**注意，这种非局部的返回只支持传给内联函数的 lambda 表达式**，这个问题后面再说把，就是`inline`、`noinline`、`crossinline`这三个关键字带来的优化以及滥用的坏处。
另外这里还有一个小 tip：如果 lambda 表达式的参数未使用，那么可以用下划线取代其名称
``` kotlin
map.forEach { (_, value) -> println("$value!") }
```

### 匿名函数
上文中的 Lambda 表达式缺少指定返回类型的能力，虽然大部分情况下返回值类型可以推导出来，但如果确实需要指定，我们可以使用**匿名函数**，
它看起来非常像一个常规函数声明，除了其名称省略了。其函数体既可以是表达式也可以是代码块：
``` kotlin
fun(x: Int, y: Int): Int = x + y
fun(x: Int, y: Int): Int {
    return x + y
}
```
如果参数类型可以推断出来，则参数类型可以省略
``` kotlin
ints.filter(fun(item) = item > 0)
```

----


对于上面的内容
函数式（SAM）接口 英文版: [Functional (SAM) interfaces](https://kotlinlang.org/docs/fun-interfaces.html)  
函数式（SAM）接口 中文版: [函数式（SAM）接口](https://book.kotlincn.net/text/fun-interfaces.html)
高阶函数和Lambda 英文版: [Higher-order functions and lambdas](https://kotlinlang.org/docs/lambdas.html)
高阶函数和Lambda 中文版: [高阶函数与 lambda 表达式](https://book.kotlincn.net/text/lambdas.html)
返回与跳转 中文版: [返回与跳转](https://book.kotlincn.net/text/returns.html)
返回与跳转 英文版: [Returns and jumps](https://kotlinlang.org/docs/returns.html)




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

未学习：
* 关键字
  * object
  * Unit
  * Nothing
  * with、let、run、apply、also
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