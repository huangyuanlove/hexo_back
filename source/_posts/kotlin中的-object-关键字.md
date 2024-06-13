---
title: kotlin中的 object 关键字
tags: [Kotlin,Android]
date: 2024-05-31 09:45:56
keywords: kotlin object,object,kotlin单例,kotlin静态方法,kotlin伴生对象,伴生对象,companion
---
kotlin 中的`object`关键字用处比较广泛，在官方文档[对象表达式与对象声明](https://book.kotlincn.net/text/object-declarations.html)有详细的介绍，比如：创建匿名对象、创建匿名内部类并继承某个类，实现多个接口、使用匿名对象作为返回和值类型、从匿名对象访问变量、单例模式、数据对象、伴生对象等，不过文章是从`对象表达式`和`对象声明`角度来区分的。
<!--more-->

### 对象表达式
对象表达式可以用来创建匿名类，就是不用`class`来声明的类，当这个类只用一次的时候是很有帮助的。我们可以从头开始定义匿名类，也可以从现有类继承，还可以实现接口。匿名类的实例也称为匿名对象，因为它们是由表达式而不是名称定义的。
#### 创建匿名类
``` kotlin
fun main() {
    val helloWorld = object {
        val hello = "Hello"
        val world = "World"
        //object expressions extend Any, so `override` is required on `toString()`
        override fun toString(): String {
            return "$hello $world"
        }
    }
    println(helloWorld)
    println(helloWorld.javaClass.simpleName)
    println(helloWorld.javaClass.name)
}
```
可以看到，这种方式在某种意义上和 js 中创建对象差不多，`helloWorld`这个实例的`helloWorld.javaClass.simpleName`是空的。当然了匿名类也是类，只是没有名字而已，当然做了继承其他类，实现其他接口。注意，同样只能单继承多实现，并且父类构造函数需要参数时可以传适当的构造参数。
比如这样：
``` kotlin
interface OnClickListener {
    fun onClick(view: View?)
}
interface  OnLongClickListener{
    fun onLongClick(view: View?)
}
abstract class OnScroll{
    abstract fun onScroll(direction: Int)
}
val viewListener = object : OnScroll(),OnLongClickListener, OnClickListener {
    override fun onScroll(direction: Int) {

    }
    override fun onClick(view: View?) {
        println("view clicked")
    }
    override fun onLongClick(view: View?) {

    }
}
```
当然也可以直接当成参数传入调用的方法，都是一样的。本质上都是创建了一个对象。


#### 使用匿名对象作为返回值或类型

当匿名对象被用作局部或私有但非内联声明（函数或属性）的类型时，其所有成员都可通过该函数或属性访问：

``` kotlin
class C {
  private fun getObject()= object {
       val x = "x"
  }
    
  private fun test(){
      println(getObject().x)
  }
}

```
如果方法供或者属性是 public 或者 private inline 时，它的实际类型可能如下：
* 如果没有明确声明类型，则是`Any`
* 如果只有一个父类或者一个接口，则是改父类或者接口类型
* 如果有一个父类和多个接口，则是方法明确返回的类型
比如
``` kotlin
class CC {
    // 返回值类型是 Any; 不能在其他方法中访问 x
    fun getObject() = object {
        val x: String = "x"
    }

    // 返回值类型是 AA; 不能在其他方法中访问 x
    fun getObjectAA() = object: AA {
        override fun funFromA() {}
        val x: String = "x"
        fun test(){
            //这里会报错，访问不到 x 属性
            println(getObject().x)
        }
    }

    // 返回值类型是 BB; 不能在其他方法中访问 x
    fun getObjectBB(): BB = object: AA, BB { // explicit return type is required
        override fun funFromA() {}
        val x: String = "x"
        fun test(){
            //这里会报错，访问不到 x 属性
            println(getObject().x)
        }
    }
}
```
#### 匿名对象访问变量
对象表达式中的代码可以访问来自包含它的作用域的变量：
``` kotlin
class AAA{
    val x = "x"
    val y = "y"
    fun getObject() = object {
        val xx = x//不报错
        val yy = y//不报错
    }
    class AAAA{
        val xx = x //报错
    }
    object BBBB{
        val xx = x//报错
    }
}

```

### 对象声明

#### 实现单例模式
在Kotlin当中，要实现`单例模式`其实非常简单，我们直接用`object`修饰类即可,当然这个单例类也是可以有父类的
``` kotlin
object MyViewListener : OnClickListener, OnLongClickListener {
    override fun onClick(view: View?) {
        println("view clicked")
    }

    override fun onLongClick(view: View?) {
        println("view long clicked")
    }
    fun test(){
        println("test")
    }
}

```
调用的时候直接`类名.方法名`即可。这里有个注意点：对象声明不能在局部作用域（即不能直接嵌套在函数内部），但是它们可以嵌套到其他对象声明或非内部类中。

#### 数据对象(data object)
当我们想要打印`object`对象声明时，字符串表示同时包含其名称和对象的哈希：
``` kotlin
object MyObject

fun main() {
    println(MyObject) // MyObject@3ac3fd8b
}
```
我们可以还用`data`关键字来修饰它
``` kotlin
data object  MyDataObject{
    val x = 3
}
```
这样的话，编译器会为这个对象生成`toString()`方法，该方法只会返回对象名字。还有成对出现的`equals()/hashCode()`.这里有一点需要注意：
被重写的`equals()`方法会将所有相同名字的`data object`都返回`true`,这个解释不是太严谨，因为绝大部分情况下，`data object`是单例的，在运行时只会有一个对象，但我们可以通过平台相关的方法创建另外一个实例对象，比如 jvm 上的反射、序列化和反序列化等。因此，在比较`data object`是否相同时，请使用`==`而不是`===`

##### data class 和 data object 的不同
虽然数据类和数据对象经常一起使用，并且有一些相同点，但有些函数在`data object`中是不会生成的
* 没有`copy()`方法，因为`data object`用作单例对象，所以不会生成该方法。如果允许创建另外个实例对象，则违反了该模式。
* 没有`componentN()`方法，该方法的一个简单的用法就是用于结构对象，允许一次性获取对象的所有属性值，并将它们作为单独的参数传递给函数或构造器。但由于`data object`没有属性，生成这些方法是没有意义的。
  
#### 伴生对象
在`kotlin`中并没有`static`关键字,那么我们如何实现静态方法的效果？我们可以使用`companion`和`object`关键字达到这个效果
``` kotlin
class  MyClassOne{
    object A{
        fun createA(): MyClassOne = MyClassOne()
    }
    companion object AA{
        fun createAA(): MyClassOne = MyClassOne()
    }
}
MyClassOne.A.createA()
MyClassOne.createAA()
```
但是看反编译之后的代码，编译器还是为我们创建了`A`和`AA`两个类。如果在`jvm`平台，我们可以使用`@JvmStatic`注解，将伴生对象的成员生成为真正的静态方法和字段。
``` kotlin
class MyClassOneJVMStatic{
    companion object AAA {
        @JvmStatic
        fun create(): MyClassOneJVMStatic = MyClassOneJVMStatic()
    }
}
```
当然，对于上面`MyClassOne`中的`AA`是可以省略名字的
``` kotlin
class MyClassTwo{
    companion object {
        fun create(): MyClassTwo = MyClassTwo()
    }
}
MyClassTwo.Companion.create()//正确，但会提示Companion是不必要的
MyClassTwo.create()//正确
```
请注意，即使伴生对象的成员看起来像其他语言的静态成员，在运行时他们仍然是真实对象的实例成员，而且，例如还可以实现接口：
``` kotlin
interface Factory<T> {
    fun create(): T
}
class MyClass {
    companion object : Factory<MyClass> {
        override fun create(): MyClass = MyClass()
    }
}
val f: Factory<MyClass> = MyClass
```

### 对象表达式和对象声明之间的语义差异

对象表达式和对象声明之间有一个重要的语义差别：

* 对象表达式是在使用他们的地方立即执行（及初始化）的。
* 对象声明是在第一次被访问到时延迟初始化的。
* 伴生对象的初始化是在相应的类被加载（解析）时，与 Java 静态初始化器的语义相匹配 。


参考：
[对象表达式与对象声明](https://book.kotlincn.net/text/object-declarations.html)
[Object expressions and declarations](https://kotlinlang.org/docs/object-declarations.html)
[静态字段](https://book.kotlincn.net/text/java-to-kotlin-interop.html#%E9%9D%99%E6%80%81%E5%AD%97%E6%AE%B5)


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
* 委托
  * <input type='checkbox' disabled='true' checked>委托类</input>
  * <input type='checkbox' disabled='true' checked>委托属性</input>
  * <input type='checkbox' disabled='true' checked>自定义委托</input>

未学习：
* 关键字
  * Unit
  * Nothing
  * inline,noinline,crossinline




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