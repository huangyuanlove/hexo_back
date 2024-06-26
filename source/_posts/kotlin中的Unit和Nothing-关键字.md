---
title: kotlin中的Unit和Nothing 关键字
tags: [Kotlin,Android]
mermaid: true
date: 2024-06-14 10:05:54
keywords:  kotlin Nothing,Unit,Void
---

让我们先从 kotlin 的类型继承关系开始：众所周知，kotlin 中所有东西都有类型，对象、函数等等，就连 Unit，Nothing 也有对应的类型。我们来看一下kotlin 中的类型层次结构。
<!--more-->

### 从顶部开始
所有类型的 Kotlin 对象都被组织成子类型/超类型关系的层次结构。该层次结构的“顶部”是抽象类Any。例如，String 和 Int 类型都是的子类型Any。
{% mermaid %}
graph BT

String-->Any
Int-->Any

{% endmermaid %}
这里的 Any 相当于 java 中的 Object，同样的，如果我们声明一个类，没有显式指定继承自哪个类，那这个类就是 Any 的直接子类。
如果该类指定了父类，那么 Any 会是该类的最终父类(祖先)
``` kotlin
open class Person(val name: String)
class Student(name:String):Person(name)
class Teacher(name:String):Person(name)
```
{% mermaid %}
graph BT
Person --> Any
Student-->Person
Teacher-->Person

{% endmermaid %}
如果一个类实现了多个接口，那么它就会有多个直接父类

``` kotlin
interface Run
interface Fly
class Bird : Run,Fly
```
{% mermaid %}
graph BT
Run --> Any
Fly --> Any
Bird-->Run
Bird-->Fly
{% endmermaid %}

Kotlin 类型检查器强制执行子类型/父类型关系，我们可以将子类型存储到超类型变量中，反过来则不行，这和 java 是一样的逻辑。
另外，kotlin 中还有可空类型，上面我们提到的类型都是非空类型，可空类型只是在可空类型后面加了一个 **?**,

``` kotlin
val name :String =null //错误：Null can not be a value of a non-null type String
val name :String? =null //正确
var s: String? = null
var s2 = "a"
s = s2
s = null
s2 = s//错误：Type mismatch.Required:String， Found:String?

```
从这个角度来看，我们可以认为(仅仅是可以认为)非空类型是对应可空类型的子类，因为非空类型可以赋值给对应的可空类型，反之则不行。
{% mermaid %}
graph BT
Any --> Any?
String? --> Any?
String-->Any
String-->String?
{% endmermaid %}

### Unit
Kotlin中的Unit即相当于Java中的void关键字，用于表示返回空结果的函数。但这里有一些不一样的地方，当 Unit 用于函数返回值时，是可以省略不写的，但kotlin 还是会认为返回了 Unit。其次，Unit 是一个真实存在的类型，并且是一个单例的。
``` kotlin

fun unitFun() {
    println("unitFun")
}
fun unitFun1(): Unit {
    println("unitFun1")
}
fun unitFun2(): Unit {
    println("unitFun2")
    return Unit
}
fun unitFun3() {
    println("unitFun3")
    return
}
fun unitFun4() {
    println("unitFun3")
    return Unit
}

public object Unit {
    override fun toString() = "kotlin.Unit"
}

```
我们发现，上面这几种写法，效果是一样的。但这里有一个点需要注意一下：
对于`unitFun2()`这个函数，跟在函数名后面的`Unit`表示返回值类型，而函数体里面的`return Unit`中的 Unit 是一个单例对象。虽然看起来什么也没有返回，但它确实返回的一个单例对象。
回到我们上面提到的可空类型上，一个比较边缘的 case:Unit?类型
``` kotlin
var nullableUnit:Unit? = null
```
这个东西我从来没有使用过，是 kotlin 类型一致性的结果

{% mermaid %}
graph BT
Any --> Any?
Unit? --> Any?
Unit-->Any
Unit-->Unit?
{% endmermaid %}

那 Unit 这个东西有什么用呢？为啥不沿用 java 中的 void ？
扔物线大佬在他的文章中给出了具体的回答：去特殊化，`Unit`去掉了无返回值的函数的特殊性，消除了有返回值和无返回值的函数的本质区别，这样很多事做起来就会更简单了。
例：有返回值的函数在重写时没有返回值
``` java
public abstract class Maker {
  public abstract Object make();
}

public class AppleMaker extends Maker {
  // 合法
  @Override
  public Apple make() {
    return new Apple();
  }
}

public class NewWorldMaker extends Maker {
  // 非法
  @Override
  public void make() {
    world.refresh();
  }
}
public class NewWorldMaker extends Maker {
  @Override
  public Object make() {
    world.refresh();
    return null;//只能去写一行 return null 来手动实现接近于「什么都不返回」的效果
  }
}
```
``` kotlin
abstract class Maker {
  abstract fun make(): Any
}

class AppleMaker : Maker() {
  override fun make(): Apple {
    return Apple()
  }
}

class NewWorldMaker : Maker() {
  override fun make() {
    world.refresh()
  }
}

abstract class Maker<T> {
  abstract fun make(): T
}

class AppleMaker : Maker<Apple>() {
  override fun make(): Apple {
    return Apple()
  }
}

class NewWorldMaker : Maker<Unit>() {
  override fun make() {
    world.refresh()
  }
}
```
例：函数类型的函数参数
``` kotlin
fun runTask(task: () -> Any) {
  when (val result = task()) {
    Unit -> println("result is Unit")
    String -> println("result is a String: $result")
    else -> println("result is an unknown type")
  }
}

...

runTask { } // () -> Unit
runTask { println("完成！") } // () -> String
runTask { 1 } // () -> Int
```


### Nothing

这里抛出一个引子：
``` kotlin
fun fail() {
    throw RuntimeException("Something went wrong")
}
```
这个方法的返回值是什么？它真的有返回值么？如果有，是什么类型？
实际上这个方法的返回值类型是`Nothing`
kotlin中有这么一个方法:`TODO`,注意，这里的 TODO 是一个方法，而不是`//todo`这一个
``` kotlin
public inline fun TODO(): Nothing = throw NotImplementedError()
```
我们来看一下 Nothing 定义
``` kotlin
/**
 * Nothing has no instances. You can use Nothing to represent "a value that never exists": for example,
 * if a function has the return type of Nothing, it means that it never returns (always throws an exception).
 */
public class Nothing private constructor()
```
这个类无法创建出任何实例，所以所有 Nothing 类型的变量或者函数，都找不到可用的值。那这个东西有啥用？正如它的注释所说， 它可以作为函数「永不返回」的提示，也就是总是抛出异常。
那这个东西有啥用？
第一个作用就是作为函数「永不返回」的提示
第二个作用就是作为泛型对象的临时空白填充，比如
``` kotlin
val emptyList: List<Nothing> = listOf()
var apples: List<Apple> = emptyList
var users: List<User> = emptyList
var phones: List<Phone> = emptyList
var images: List<Image> = emptyList
```
既省事，又省内存。当然也可以用在 Set、Map 中
第三个作用：语法的完整化
Nothing 的「是所有类型的子类型」这个特点，还帮助了 Kotlin 语法的完整化。在 Kotlin 的下层逻辑里，throw 这个关键字是有返回值的，它的返回值类型就是 Nothing。虽然说由于抛异常这件事已经跳出了程序的正常逻辑，所以 throw 返回不返回值、返回值类型是不是 Nothing 对于它本身都不重要，但它让类似 `TODO()`函数的写法成为了合法的。

{% mermaid %}
graph BT
String --> Any
Int --> Any
Unit --> Any
Person --> Any
Nothing --> Person
Nothing --> Unit
Nothing --> Int
Nothing --> String
{% endmermaid %}


参考  
[A Whirlwind Tour of the Kotlin Type Hierarchy](http://natpryce.com/articles/000818.html)
[Unit 为啥还能当函数参数？面向实用的 Kotlin Unit 详解](https://rengwuxian.com/kotlin-unit/)
[这玩意真的有用吗？对，是的！Kotlin 的 Nothing 详解](https://rengwuxian.com/kotlin-nothing/)
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
* 委托
  * <input type='checkbox' disabled='true' checked>委托类</input>
  * <input type='checkbox' disabled='true' checked>委托属性</input>
  * <input type='checkbox' disabled='true' checked>自定义委托</input>

未学习：
* 关键字
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