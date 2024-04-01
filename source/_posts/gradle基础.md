---
title: gradle基础
tags: [Gradle,Android]
date: 2018-11-11 17:21:04
keywords: gradle基础,Android构建脚本
---

参考自《Android Gradle权威指南》

先来回顾一下groovy的一些基础语法

1. 调用方法的时候，圆括号是可以省略的，比如

   ``` groovy
   def method1(int a, int b){
       println a+b
   }
   
   task invokeMethod <<{
   	method1(1,2)
       method1 1,2
   }
   ```

2. 定义方法时，return是可以不写的，比如

   ``` groovy
   def getMaxNumber(int a, int b){
       if(a>b){
           a
       }else {
           b
       }
   }
   task printMethodReturn <<{
       def max1 = getMaxNumber 1,2
       def max2 = getMaxNumber 3,5
       println "max1 :${max1},max2:${max2}"
   }
   ```

   执行`gradle printMethodReturn`可以看到输出  `max1 :2,max2:5`

3. 代码块是可以作为参数传递的，比如：

   ``` groovy
   //最直接的写法
   numList.each({println it})
   //groovy规定，如果方法的最后一个参数是闭包，可以放大方法外面
   numList.each(){
       println it
   }
   // 还规定了，如果出现上面的情况，圆括号也是可以省略的
   numList.each{
       println it
   }
   ```

   是不是和python很像。

   <!--more-->

   #### JavaBean

   在Java中为了获取、修改属性的值，通常需要通过生成`getter/setter`方法，但是在`groovy`中我们可以这样

   ``` groovy
   class Person{
       private String name
   
       public int getAge(){
           return 12
       }
   }
   
   task PersonField <<{
       Person p  = new Person()
       println "name is ${p.name}"
       p.name = "张三"
       println "name is ${p.name}"
       println "age is ${p.age}"
   }
   ```

   执行该任务，得到输出

   > name is null
   > name is 张三
   > age is 12

   在没有给name属性赋值时，输出为null，赋值之后，输出给定的值。并且并不需要通过getter/setter方法来进行操作，看起来有点像kotlin。

   对于age来讲，我们并没有在对象中声明该属性，也就是说并不一定需要定义成员变量才能作为类的属性访问。

   #### 闭包

   闭包其实就是一段代码块，我们以集合的each方法为例

   ``` groovy
   def customClosure(closure){
       for(i in 1..10){
           closure(i)
       }
   }
   
   task testClosure <<{
       customClosure{
           print it
       }
   }
   ```

   执行该任务，可以得到1到10的打印输出，可以将上述代码简单粗暴的理解为

   ``` groovy
   def customClosure(){
       for(i in 1..10){
           print i
       }
   }
   
   task testClosure <<{
       customClosure()
   }
   ```

   ##### 闭包的参数

   上面的例子中只用到了一个参数，用于接收一个闭包，默认就是it，当有多个参数时，需要把参数一一列出

   ``` groovy
   task helloClosure <<{
       eachMap{k,v -> 
           println "${k} is ${v}"
       }
   }
   
   def eachMap(closure){
       def map1=["name":"张三","age":18]
       map1.each{
           closure(it.key,it.value)
       }
   }
   ```

   执行helloClosure任务，得到

   > $ gradle -q helloClosure
   > name is 张三
   > age is 18

   ##### 闭包委托

   groovy的闭包委托有thisObject、owner和delegate三个属性，当在闭包内调用方法时，	由他们来确定使用哪个对象来处理，默认情况owner和delegate是相等的，但是delegate属性可以被修改

   ``` groovy
   //闭包委托
   def method1(){
       println "Context this :${this.getClass()} in root"
       println "method1 in root"
   }
   
   class Delegate{
       def method1(){
           println "Delegate this : ${this.getClass()} in Delegate"
           println "method1 in Delegate"
       }
   
       def test(Closure<Delegate> closure){
               closure(this)
       }
   }
   
   task helloDelegate <<{
       new Delegate().test{
           println "thisObject :${thisObject.getClass()}"
           println "owner:${owner.getClass()}"
           println "delegate:${delegate.getClass()}"
           method1()
           it.method1()
       }
   }
   ```

   执行任务可以得到

   > $ gradle -q helloDelegate
   > thisObject :class build_apnm7l2hrtoaa0bd51bpb5xcf
   > owner:class build_apnm7l2hrtoaa0bd51bpb5xcf$_run_closure4
   > delegate:class build_apnm7l2hrtoaa0bd51bpb5xcf$_run_closure4
   > Context this :class build_apnm7l2hrtoaa0bd51bpb5xcf in root
   > method1 in root
   > Delegate this : class Delegate in Delegate
   > method1 in Delegate

   从上面例子可以看到，在不进行任何修改的情况下，thisObject的优先级最高，默认情况下优先使用thisObject来处理闭包中的方法，如果有则执行。从输出中可以看到这个thisObject就是构建脚本的上下文，它和脚本中的this是一样的。

   在DSL中，比如gradle，一般会指定delegate为当前的it，这样我们就可以在闭包中对该it进行配置，或者调用其方法：

   ``` groovy
   class Person {
       String personName
       int personAge
   
       def dumpPerson(){
           println "name is ${personName}, age is ${personAge}"
       }
   }
   
   def person(Closure<Person> closure){
       Person p = new Person()
       closure.delegate = p
       //委托模式优先
       closure.setResolveStrategy(Closure.DELEGATE_FIRST)
       closure(p)
   }
   
   task configClosure << {
       person{
           personName="张三"
           personAge = 20
           dumpPerson()
       }
   }
   ```

   上面实例中设置了	委托对象为当前创建的Person实例，并且设置了委托模式优先，所以我们使用person方法创建实例的时候，可以在闭包里面直接对该实例进行配置。

   ----

   以上，