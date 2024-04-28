---
title: Kotlin中的函数类型及扩展函数
tags: [Kotlin,Android]
date: 2024-04-22 17:28:13
keywords: Kotlin,扩展函数,扩展属性,函数接收者,带有接收者的函数类型
---

继续kotlin 的学习，之前只是学了点皮毛中的皮毛，会了一些简单语法而已。最后面列了一个大纲，认真的学习一下。
今天的内容是**扩展**。gradle：8.5，插件：id 'org.jetbrains.kotlin.jvm' version '1.9.23'
<!-- more -->


### 简介和使用
kotlin 中扩展可以给已有的类添加额外的方法和属性，看起来就像是修改了类的源码一样，而不是像 java 一样需要继承该类然后添加自己的方法。扩展又分为扩展函数和扩展属性。
那么如何使用嘞？其实和声明普通函数几乎一致，只是多了一个叫做"接收者"的东西，也就是文档中的Receiver，说白了，其实就是限制这个接收者类型才能使用这个方法，也就是我们要对这个类型 **"添加"** 一个方法。
比如我们想要给字符串类型添加一个获取最后一个元素的方法：

``` kotlin
fun String.lastChar(): Char?{
    if(this.isEmpty()){
        return null
    }
    return get(length - 1)
}
```
上面的方法声明表示：对`String`类型定义一个无参的`lastChar`方法，返回值是`Char?`，使用的时候就像使用 String 类中的方法一样使用就好了：
``` kotlin
fun main() {
    val s = "kotlin"
    println(s.lastChar())
}
```
那么扩展属性怎么使用嘞？和扩展函数差不多：
``` kotlin
val String.firstChar:Char?
    get() = if(this.isEmpty()) null else get(0)
```
可以简单的认为上面的声明是这样:对`String`类型顶一个`firstChar`属性，类型是`Char?`,使用时和使用 String 类中的属性一样就好了:
``` kotlin
fun main() {
    val s = "kotlin"
    println(s.lastChar())
    println(s.firstChar)
}
```

### 思考：Java中如何使用？

接下来思考一下在 java 中如何调用嘞？得先看看 kotlin 是如何实现扩展的。最简单的方法，看反编译后的字节码文件：
顶部菜单中 tools-->kotlin-->Show Kotlin Bytecode，然后点Decompile就可以看到了
![decompile kotlin bytecode](image/kotlin/show_kotlin_bytecode.png)
代码大致如下
``` Java
public final class MainKt {
   @Nullable
   public static final Character lastChar(@NotNull String $this$lastChar) {
      Intrinsics.checkNotNullParameter($this$lastChar, "<this>");
      return ((CharSequence)$this$lastChar).length() == 0 ? null : $this$lastChar.charAt($this$lastChar.length() - 1);
   }

   @Nullable
   public static final Character getFirstChar(@NotNull String $this$firstChar) {
      Intrinsics.checkNotNullParameter($this$firstChar, "<this>");
      return ((CharSequence)$this$firstChar).length() == 0 ? null : $this$firstChar.charAt(0);
   }

   public static final void main() {
      String s = "kotlin";
      Character var1 = lastChar(s);
      System.out.println(var1);
      var1 = getFirstChar(s);
      System.out.println(var1);
   }

   // $FF: synthetic method
   public static void main(String[] args) {
      main();
   }
}
```
可以看到反编译之后的代码只是添加了两个静态方法而已，这样的话，在 Java 中我们就可以这么使用了
```Java
public class TestMain {
    public static void main(String[] args) {
        String s = "Hello World!";
        System.out.println( MainKt.lastChar(s));
        System.out.println( MainKt.getFirstChar(s));
    }
}
```
### 思考：作用域，继承与重载

接下来思考另外一个问题：作用域，或者说我们可以在哪里声明、在哪里调用扩展函数？
上面的例子中都是声明为了顶级函数(top level),我们可以在任意地方使用对应的类型进行调用，如果声明在类里面会怎么样？
``` kotlin
class Example{
    fun String.isEmail():Boolean {
        return this.contains("@")
    }

    fun test(){
        val email = "a@a.com"
        println(email.isEmail())
    }

}

fun test(){
    val email = "a@a.com"
    println(email.isEmail())
}
```
可以看到在类里面定义的扩展函数，只能在类里面调用，在类外是无法使用的。但是，我们可以在继承Example的类中使用，比如这样
``` Kotlin
class SubExample :Example(){
    fun subTest(){
        val email = "a@a"
        println(email.isEmail())
    }
}
```
那么问题来了，如果对`Example`定义一个扩展函数，那么在子类SubExample中能调用么？答案是可以的，但是不能覆写，因为kotlin中的函数默认是`final`不能被覆写的，同时定义扩展函数时又不能被`open`修饰，从语法上讲，这是扩展函数不能被覆写的原因。看反编译之后的代码，定义为顶级函数的扩展函数是 static 的，因此也不能被覆写。
那么在 Java 中能不能用嘞？遇事不决先看反编译后的代码
``` Java
public class Example {
   public final boolean isEmail(@NotNull String $this$isEmail) {
      Intrinsics.checkNotNullParameter($this$isEmail, "<this>");
      return StringsKt.contains$default((CharSequence)$this$isEmail, (CharSequence)"@", false, 2, (Object)null);
   }

   public final void test() {
      String email = "a@a.com";
      boolean var2 = this.isEmail(email);
      System.out.println(var2);
   }
}
```
这样的话就可以通过`Example`实例来调用了。同样注意到在`Example`类中定义的扩展函数`isEmail`被 final 修饰了，因此也无法通过继承来覆写该方法。

### 思考：扩展函数如何引用？
嘿嘿嘿,我们知道函数是可以通过双冒号引用的

``` kotlin
fun sayHi(name: String) {
    println("hi $name")
}


fun referenceMethod(method: (String) -> Unit) {
    method("xuan")
    method.invoke("yuan")
}
fun main() {
    referenceMethod(::sayHi)
    referenceMethod { name -> println("hello $name") }
}
```
那么扩展函数应该如何引用嘞？这里先学怎么用，后面再学函数类型吧
如果我们将扩展函数定义为顶级函数，那么在应用的时候和引用这个类本身的成员函数没啥区别,比如在一开始我们对 String 定义的扩展函数 lastChar,我们可以这么引用：
``` Kotlin
val lastCharFun1 = String::lastChar
```
但是，如果我们将扩展函数定义在类里面又该如何应对？应对不了，没法引用。
为什么？思考一个问题，扩展函数属于哪个类？实际上可以认为扩展函数谁都不属于，只是加了一个限定，限定哪个类型的对象可以调用这个函数。
另外一个问题，语法上引用类的成员函数是类名双冒号函数名，那引用扩展函数也是这样，但是把扩展函数定义在其他类中，我们应该用哪个类名？干脆不能引用就好了。

----
### 总结
了解了扩展函数、扩展属性 及其作用域。了解了在 Java 层面如何实现的以及 Java 中如何使用。翻看 kotlin 源码，有很多都是基于扩展来实现的，比如 String、比如一些数字类型 Float、Double 等。

到此，扩展就差不多了，应该还会有一些小细节上的问题，但问题应该不大。接下来应该会学习一下遗留下来的问题：函数类型以及lambda 表达式

---- 
已学习：
* 扩展
  * <input type='checkbox' disabled='true' checked>扩展函数</input>
  * <input type='checkbox' disabled='true' checked>扩展属性</input>
  * <input type='checkbox' disabled='true' checked>作用域</input>

未学习：
* Lambda表达式
    * SAM 转换
    * 函数类型
  
* 函数类型
    * 带有接收者的函数类型

* 关键字
  * object
  * Unit
  * Nothing
  * with、let、run、apply、also


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
  

