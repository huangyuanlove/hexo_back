---
title: kotlin委托
tags: [Kotlin,Android]
date: 2024-05-27 10:30:07
keywords: Kotlin,Kotlin委托,委托属性,属性委托,委托类,Delegation,Delegated properties
---
经常在 Kotlin 的源码三方库中看到`by`关键字，这种写法就是委托，主要有两个应用场景，一个是委托类，另一个是委托属性，每个场景中又有不同的用法，我们可以对比 Java 的委托来学习 Kotlin 的委托。
<!--more-->

### 委托(类委托、接口委托)
其实我们在 Java 和 Android 中经常会用到委托，
``` Java
public class Delegated {

    interface Base{
        void print();
    }
    class BaseImpl implements Base{
        private final int x;

        public BaseImpl(int x) {
            this.x = x;
        }

        @Override
        public void print() {
            System.out.println(x);
        }
    }

    class Derived implements Base{
        private final Base base;

        public Derived(Base base) {
            this.base = base;
        }

        @Override
        public void print() {
            base.print();
        }
    }
}

```
我们有一个接口`Base`，一个实现类`BaseImpl`。假如我们想要在实现类中添加一些方法，但又不想重新写一遍接口实现，第一种我们可以继承`BaseImpl`,另外一种就是实现接口`Base`，传入一个实现类的实例，将所有的接口请求都交给实现类的实例来完成。
虽然官方说**委托模式已经证明是实现继承的一个很好的替代方式(The Delegation pattern has proven to be a good alternative to implementation inheritance)**,但选择权还是在大家手上，看情况而定，没有银弹。
那么在 kotlin 中应该怎么写呢？如果我们用 java 的思想来写，无非就是换换关键字，然后一坨模板代码，其实在 kotlin 中是可以通过`by`关键字零模板代码支持的
``` kotlin
interface Base {
    fun print()
}

class BaseImpl(val x: Int) : Base {
    override fun print() {
        println(x)
    }
}

class Derived(b: Base) : Base by b
```
`Derived`的超类型列表中的`by`子句表示`b`将会在`Derived`中内部存储， 并且编译器将生成转发给`b`的所有`Base`的方法。
这个就是 kotlin 中的委托，有的地方也叫委托类或者委托接口。
这里有一点需要注意下，覆盖(override)是符合预期的：编译器会使用 override 覆盖的实现而不是委托对象中的.
``` kotlin
interface Base {
    fun printMessage()
    fun printMessageLine()
}

class BaseImpl(val x: Int) : Base {
    override fun printMessage() { println(x) }
    override fun printMessageLine() { println(x) }
}

class Derived2(b: Base) : Base by b {
    override fun printMessage() { println("abc") }
}

fun main() {
    val b = BaseImpl(10)
    Derived2(b).printMessage()
    Derived2(b).printMessageLine()
}
```
我们在`Derived2`中覆写了`printMessage`这个方法，那么在调用的时候，就是用的我们覆写的方法。

### 属性委托
``` kotlin
class Example {
    var p: String by Delegate()
}
```
语法是：`val/var <属性名>: <类型> by <表达式>`。在`by`后面的表达式是该`委托`， 因为属性对应的`get()`（与`set()`）会被委托给它的`getValue()`与`setValue()`方法。 属性的委托不必实现接口，但是需要提供一个`getValue()`函数（对于`var`属性还有`setValue()`）。
先从最简单的委托开始，最后再看自定义委托。

#### 标准委托
借用官网的一个例子
``` kotlin
class MyClass {
   var newName: Int = 0
   @Deprecated("Use 'newName' instead", ReplaceWith("newName"))
   var oldName: Int by this::newName
}
fun main() {
   val myClass = MyClass()
   // 通知：'oldName: Int' is deprecated.
   // Use 'newName' instead
   myClass.oldName = 42
   println(myClass.newName) // 42
}
```
这是一种最简单的委托方式。通过查看对应的 java 代码，发现其实就是对同一个成员变量的读写
``` Java
public final class MyClass {
   private int newName;

   public final int getNewName() {
      return this.newName;
   }

   public final void setNewName(int var1) {
      this.newName = var1;
   }

   /** @deprecated */
   public final int getOldName() {
      return this.newName;
   }

   /** @deprecated */
   public final void setOldName(int var1) {
      this.newName = var1;
   }
}
```
`MyClass`中的四个方法都是对`newName`这个字段的读写。

除此之外，委托属性可以是：
* 顶层属性
* 同一个类的成员或扩展属性
* 另一个类的成员或扩展属性

比如
``` kotlin
var topLevelInt: Int = 0
class ClassWithDelegate(val anotherClassInt: Int)

class MyClass(var memberInt: Int, val anotherClassInstance: ClassWithDelegate) {
    var delegatedToMember: Int by this::memberInt//同一个类的成员
    var delegatedToTopLevel: Int by ::topLevelInt//顶层属性

    val delegatedToAnotherClass: Int by anotherClassInstance::anotherClassInt//另一个类的成员
}
var MyClass.extDelegated: Int by ::topLevelInt//顶层属性
```
这种委托方式在我们做版本升级修改字段时是挺常用的,将旧字段委托给新字段，并将旧字段标记为过时。

#### 懒加载委托
这种方式就是当我们首次访问这个属性的时候才会去初始化这个属性，从而避免不必要的资源消耗，和我们用 java 写单例模式的懒加载是一样的。
只会在首次访问的时候初始化这个属性，然后缓存起来，下次访问时直接返回。
``` kotlin
val lazyValue: String by lazy {
    println("computed!")
    "Hello"
}

fun main() {
    println(lazyValue)
    println(lazyValue)
}
```
这里的 lazy 是一个高阶函数：
``` kotlin
public actual fun <T> lazy(initializer: () -> T): Lazy<T> = SynchronizedLazyImpl(initializer)

public actual fun <T> lazy(mode: LazyThreadSafetyMode, initializer: () -> T): Lazy<T> =
    when (mode) {
        LazyThreadSafetyMode.SYNCHRONIZED -> SynchronizedLazyImpl(initializer)
        LazyThreadSafetyMode.PUBLICATION -> SafePublicationLazyImpl(initializer)
        LazyThreadSafetyMode.NONE -> UnsafeLazyImpl(initializer)
    }
```
可以看到，lazy接收一个 mode 参数，如果没有传入的话，默认是`SynchronizedLazyImpl`线程安全的：该值只在一个线程中计算，但所有线程都会看到相同的值。如果初始化委托的同步锁不是必需的，这样可以让多个线程同时执行，那么将`LazyThreadSafetyMode.PUBLICATION`作为参数传给 lazy()。
如果我们确定初始化将总是发生在与属性使用位于相同的线程， 那么可以使用`LazyThreadSafetyMode.NONE`模式。它不会有任何线程安全的保证以及相关的开销。
所以这个参数的选择也要看具体应用场景。

#### 可观察委托
如果我们想要观察属性值的变化，可以使用`Delegates.observable()`，它接受两个参数：初始值与修改时处理程序（handler）。
``` kotlin
class  ObservableItem{
    var name :String by Delegates.observable("initialValue"){
        prop,old,new->
        println("$prop  $old -> $new")
    }
}
```
当我们给`name`赋值的时候，就会触发传入的处理程序，但这里我们只能观察到赋值，但并不能做拦截，如果想要截获取值并**否决**,可以使用`vetoable()`

#### 可否决委托
如果我们想在观察属性值变化的同时决定是否使用新的值，可以使用`Delegates.vetoable`,同样的，它也接受两个参数：它接受两个参数：初始值与修改时处理程序（handler）。只不过这里的 handler 需要返回一个布尔值，告诉程序是否使用新值。
``` kotlin
class VetoableItem{
    var name :String by Delegates.vetoable("initialValue"){
        prop,old,new->
        println("$prop  $old -> $new")
        new.length > 3
    }
}

fun main(){
    val vetoableItem = VetoableItem()
    println(vetoableItem.name)
    vetoableItem.name = "123"
    println(vetoableItem.name)
    vetoableItem.name = "1234"
    println(vetoableItem.name)
}
//输出
// initialValue
// var com.huangyuanlove.VetoableItem.name: kotlin.String  initialValue -> 123
// initialValue
// var com.huangyuanlove.VetoableItem.name: kotlin.String  initialValue -> 1234
// 1234
```
在这里，只有当`new`的长度大于 3 时，我们才会将`new`赋值给`name`，

#### 将属性储存在映射中
一个常见的用例是在一个映射（map）里存储属性的值。 这经常出现在像解析 JSON 或者执行其他“动态”任务的应用中。 在这种情况下，你可以使用映射实例自身作为委托来实现委托属性。
``` kotlin
class MapItem(map: Map<String,Any?>){
    val name: String by map
    val age:Int by map
    val address:String by map
}
fun main(){
  val map:Map<String,Any?> = mapOf(
    "name" to "xuan",
    "age" to 18,
  )
  val mapItem = MapItem(map)
  println("${mapItem.name}  ${mapItem.age}")
  println("${mapItem.name}  ${mapItem.age}  ${mapItem.address}")
}
```
这里需要注意，假如我们传入的`map`里面没有对应属性，当程序运行时，这个属性没有被使用是没问题的，比如上面打印`name`和`age`。但是当我们使用这个属性的时候，就是上面打印`address`,会抛出异常`Key address is missing in the map.`.另外一方面，我们将传入的 map 的值声明为了可空，这就意味着在调用出传入了空值，比如`"address" to null,`,代码是可以运行的，但对`address`这个属性做处理的时候会报空指针异常，这些都是需要额外注意的地方。
还有一点需要注意，如果是对于`var`属性，需要将`Map`替换成`MutableMap`,但是这样的话它们两个可是双向绑定的哟，比如下面这种
``` kotlin
class MutableMapItem(map:MutableMap<String,Any?>){
    var name: String by map
    var age: Int by map
    var address: String by map
}
fun main() {
      val map:MutableMap<String,Any?> = mutableMapOf(
        "name" to "xuan",
        "age" to 18,
        "address" to "beijing"
    )
    val mutableMapItem = MutableMapItem(map)
    println("${mutableMapItem.name}  ${mutableMapItem.age} ${mutableMapItem.address}")
    println(map)
    mutableMapItem.name = "huang"
    println("${mutableMapItem.name}  ${mutableMapItem.age} ${mutableMapItem.address}")
    println(map)
    mutableMapItem.name = "yuan"
    println("${mutableMapItem.name}  ${mutableMapItem.age} ${mutableMapItem.address}")
    println(map)
}
//输出
// xuan  18 beijing
// {name=xuan, age=18, address=beijing}
// huang  18 beijing
// {name=huang, age=18, address=beijing}
// yuan  18 beijing
// {name=yuan, age=18, address=beijing}
```

#### 局部属性委托
可以将局部变量声明为委托属性。 例如，你可以使一个局部变量惰性初始化：
``` kotlin
fun example(computeFoo: () -> Int) {
    val memoizedFoo by lazy(computeFoo)
    val someCondition = false

    if (someCondition && memoizedFoo>0 ) {
        println(memoizedFoo+1)
    }
}
```

`memoizedFoo`变量只会在第一次访问时计算。 如果`someCondition`失败，那么该变量根本不会计算。

#### 自定义委托
先看一下自定义委托的要求有哪些，示例是这样的
``` kotlin
class Resource

class Owner {
    var varResource: Resource by ResourceDelegate()
}

class ResourceDelegate(private var resource: Resource = Resource()) {
    operator fun getValue(thisRef: Owner, property: KProperty<*>): Resource {
        return resource
    }
    operator fun setValue(thisRef: Owner, property: KProperty<*>, value: Any?) {
        if (value is Resource) {
            resource = value
        }
    }
}
```
总结一下
1. 对于`var`修饰的属性，我们必须要有`getValue`、`setValue`这两个方法，同时，这两个方法必须用`operator`关键字修饰。
2. 由于`varResource`是`Owner`,因此`getValue`、`setValue`这两个方法中的`thisRef`的类型，必须要是`Owner`或者是`Owner的父类`。一般来说，这三处的类型是一致的，当我们不确定委托属性会处于哪个类的时候，就可以将`thisRef`的类型定义为`Any?`。
3. 由于委托的属性是`Resource`类型，那么对于自定义委托中的`getValue`、`setValue`参数及返回值需要是`String类型或者是它的父类`

我们可以把上面的代码当成模板代码，都是这样写就好了。如果觉得麻烦，可以使用标准库中的接口`ReadOnlyProperty`和`ReadWriteProperty`将委托创建为匿名对象，而无需创建新类。它们提供所需的方法:`getValue()`在`ReadOnlyProperty`中声明；`ReadWriteProperty`扩展了它并添加了`setValue()`。这意味着可以在需要`ReadOnlyProperty`时传递 `ReadWriteProperty`。
比如像这样
``` kotlin
fun resourceDelegate(resource: Resource= Resource()) :ReadWriteProperty<Owner,Resource> =
    object:ReadWriteProperty<Owner,Resource>{
        var curValue = resource
        override fun getValue(thisRef: Owner, property: KProperty<*>): Resource=curValue
        
        override fun setValue(thisRef: Owner, property: KProperty<*>, value: Resource) {
            curValue = value
        }

    }

class Owner {
    val readOnlyResource: Resource by resourceDelegate()  // ReadWriteProperty as val
    var readWriteResource: Resource by resourceDelegate()
}

```
#### 提供委托
通过定义 provideDelegate 操作符，可以扩展创建属性实现所委托对象的逻辑。 如果 by 右侧所使用的对象将 provideDelegate 定义为成员或扩展函数， 那么会调用该函数来创建属性委托实例。比如在初始化之前检查一致性。
``` kotlin
class ResourceDelegate<T> : ReadOnlyProperty<MyUI, T> {
    override fun getValue(thisRef: MyUI, property: KProperty<*>): T { ... }
}

class ResourceLoader<T>(id: ResourceID<T>) {
    operator fun provideDelegate(
            thisRef: MyUI,
            prop: KProperty<*>
    ): ReadOnlyProperty<MyUI, T> {
        checkProperty(thisRef, prop.name)
        // 创建委托
        return ResourceDelegate()
    }

    private fun checkProperty(thisRef: MyUI, name: String) { …… }
}

class MyUI {
    fun <T> bindResource(id: ResourceID<T>): ResourceLoader<T> { …… }

    val image by bindResource(ResourceID.image_id)
    val text by bindResource(ResourceID.text_id)
}
```
`provideDelegate`的参数与`getValue`的相同：

* `thisRef`必须与`属性所有者`类型（对于扩展属性必须是被扩展的类型）相同或者是它的超类型；
* `property`必须是类型`KProperty<*>`或其超类型。



----

参考：

[委托](https://book.kotlincn.net/text/delegation.html)
[Delegation](https://kotlinlang.org/docs/delegation.html)

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
* 委托
  * <input type='checkbox' disabled='true' checked>委托类</input>
  * <input type='checkbox' disabled='true' checked>委托属性</input>
  * <input type='checkbox' disabled='true' checked>自定义委托</input>

未学习：
* 关键字
  * object
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