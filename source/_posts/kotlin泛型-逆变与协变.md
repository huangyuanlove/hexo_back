---
title: kotlin泛型:逆变与协变
tags: [Kotlin,Android]
date: 2024-04-30 10:59:38
mermaid: true
keywords: kotlin 泛型,逆变,Covariant,协变,Variance,使用处型变,声明处型变,星投影,Star-Projections
---
泛型中涉及到的概念也不少,型变(Variance)、逆变(Contravariance)、协变(Covariance)、不变(Invariant).在 kotlin 中还有三个关键字`in`、`out`、`where`、`reified`等,在java中同样也有`? extends`、`? super`、`?`
这些概念是啥意思嘞？引用点概念说明
> 型变(Variance)、协变(Covariance)、逆变(Contravariance)和不变(Invariant)是相关但不同的概念.

>型变是指泛型类型参数在子类型关系中的行为.它描述了一个泛型类型是否允许类型参数的子类型关系与泛型类型参数的子类型关系保持一致.在泛型中,可以有三种型变类型：协变、逆变和不变.

>协变是指如果一个泛型类型的子类型关系与其类型参数的子类型关系保持一致,则该泛型类型是协变的.简而言之,如果子类型的泛型参数是父类型泛型参数的子类型,就可以说该泛型类型是协变的.

>逆变是指如果一个泛型类型的子类型关系与其类型参数的子类型关系相反,则该泛型类型是逆变的.简而言之,如果子类型的泛型参数是父类型泛型参数的超类型,就可以说该泛型类型是逆变的.

>不变是指一个泛型类型的子类型关系与其类型参数无关,即类型参数的子类型关系与泛型类型的子类型关系无关.在不变的情况下,不能将父类型的对象赋值给子类型的对象,也不能将子类型的对象赋值给父类型的对象.

>因此,可以说协变和逆变是型变的两种具体形式,而不变则是型变的一种特殊情况.

>总结起来,协变、逆变和不变描述了泛型类型参数与泛型类型之间子类型关系的不同行为.协变和逆变是对子类型关系的具体约束,而不变则是没有任何子类型关系的约束.它们之间是互相排斥的关系,不是包含关系.

一脸懵了吧😳?问题不大,结合代码具体看一下就差不多了

我们还是结合 java 和 kotlin 对比来看一下

### Java中的泛型
我们先用Java代码来看一下,假如我们有如下三个类：
``` java
public class Animal {
}

public class Dog extends Animal {
}

public static class Poodle extends Dog {
}

public class Bird extends Animal {
}
```
由于 java 的多态性,我们可以这么写
``` java
Animal animalDog = new Dog();
Animal animalBird = new Bird();
ArrayList<Animal> animalList = new ArrayList<>();
animalList.add(new Bird());
animalList.add(new Dog());
```
这么写是没问题的,我们可以把子类添加到父类列表中,但当我们在`animalList`中获取数据时返回的是`Animal`类型,如果用到子类的特性,还需要使用`instanceof`来判断一下类型.
但如果我们这么写是不行的
``` Java
ArrayList<Animal> animalList = new ArrayList<Dog>();
```
因为 java 的泛型具有不变性,在Java 里面认为`ArrayList<Animal>`和`ArrayList<Dog>`没啥关系.
同样的,当我们想要用方法重载时也会遇到这种情况
``` Java
public void animal(ArrayList<Dog> dogs){
    
}
public void animal(ArrayList<Bird> birds){
    
}
```
如果我们这么写的话会报错,IDE 会提示我们相应的信息
> animal(ArrayList<Dog>)' clashes with 'animal(ArrayList<Bird>)'; both methods have same erasure

两个方法的参数有相同的擦除类型,编译后会被认为是同一个方法.
同样的,我们在**捕获泛型异常**时也会有类似的报错信息.

### Java中的泛型擦除
面试常见的八股文,我们来复习一下,这部分可以跳过不看.
比如我们在C#中有如下代码
``` C#
using System; 

public class Program{ 
    public static void Main(String[] args){ 
        test<string>(); 
    }    
    public static void test<T>(){ 
        Console.WriteLine(typeof(T)); 
    } 
} 
```
这里的泛型 T 类型string 是可以在运行时获取到的,并且在这里是一个真实可用的类型.
但在Java是不行的,由于向上兼容历史代码的原因 Java 采用了`Code sharing`的策略,使得泛型只存在于源码阶段,编译过后的Class文件并不存在泛型,虚拟机并不知道泛型的存在,所以说Java中的泛型是一种伪泛型,这种参数类型只存在于源码阶段在编译后并不存在的机制我们叫做**泛型擦除**.为了保持泛型继承或实现关系的正确性,java 中还有一种策略：**桥方法生成(Bridge Method Generation)**:
一个简单的例子来说明桥方法生成：
``` Java
class Shape<T> {
    public void draw(T shape) {
        System.out.println("Drawing shape: " + shape.toString());
    }
}

class Circle extends Shape<String> {
    @Override
    public void draw(String shape) {
        System.out.println("Drawing circle: " + shape);
    }
}
```
在类型擦除后,编译器会生成桥方法来保持泛型继承关系的正确性.在这个示例中,编译器会生成一个桥方法,使得Circle类的方法签名与父类的方法签名保持一致,但返回类型被擦除为父类的类型参数.
``` Java
class Circle extends Shape<String> {
    @Override
    public void draw(String shape) {
        System.out.println("Drawing circle: " + shape);
    }

    // 生成的桥方法
    @Override
    public void draw(Object shape) {
        draw((String) shape);
    }
}
```
通过生成的桥方法,即draw(Object shape),在类型擦除后仍然能够正确地调用泛型方法.这样,即使在编译器看不到具体的泛型类型信息,仍然可以通过桥方法来调用正确的方法实现.
感兴趣的话可以搜一下关键字:泛型擦除、桥方法生成、Code sharin、Code specialization

### Java 中的泛型通配符

假如我们真的有像上面那种赋值需求怎么搞？java 给我们提供了`泛型通配符`: **? extends** 和 **? super** 来解决这个问题.
啰嗦一下：在继承关系上,一般情况下将父类放在上方,子类放在下方.比如上面定义的类
{% mermaid %}
graph TB

A(Animal)
A10(Dog)
A11(Bird)
A20(Poodle)
A-->A10
A-->A11
A10-->A20

{% endmermaid %}


#### ? extends
我们可以这么写
``` Java
ArrayList<? extends Animal> arrayList ;
arrayList = new ArrayList<Bird>();
arrayList = new ArrayList<Dog>();
arrayList = new ArrayList<Animal>();
```
这里的`? extends`叫做**上界通配符**,可以使 Java 泛型具有**协变性 Covariance**,协变就是允许上面的赋值是合法的.
不过这里的`extends`和我们定义`class`时继承某个类用的`extends`有一点点不一样,除了上界所有的直接子类、间接子类还包含它本身,并且上界也可以是 interface.
在上面的例子中,`ArrayList<? extends Animal>`表示列表中可以存放 Animal 及其子类、间接子类的类型.也就是确认了它的上限能到哪一层.
但我们在使用的泛型通配符之后,在使用上会有一些小问题：
``` Java
arrayList.add(new Dog());//error
arrayList.add(new Bird());//error
arrayList.add(new Animal());//error
Animal result =  arrayList.get(0);//ok
```

由于`arrayList`中存放的可以是**Animal 及其子类、间接子类的类型**,所以我们并不确定是哪种类型,因此我们无法向列表中添加元素,但可以确定的是,将列表中的元素赋值给 Animal类型的变量是没问题的.
像这种只能从列表中读取数据提供,但不能向列表中写入的情况我们称之为`生产者`

#### ? super

我们可以这么写
``` Java
ArrayList<? super Dog> list ;
list = new ArrayList<Dog>();
list = new ArrayList<Animal>();
list = new ArrayList<Poodle>();//error

```
这里的`? super`叫做**下界通配符**,可以使Java泛型具**逆变性 Contravariance**,逆变就是允许上面的赋值是合法的.
通过代码我们可以看到下界通配符确定了列表的下限,也就是确认了下限在哪一层,我们可以将该层及以上的类型赋值给 list.同样的,我们在使用上也有一点点小问题：
``` Java
list.add(new Dog());
list.add(new Poodle());
list.add(new Animal());//error

Object dog = list.get(0);
```
因为list 中存放的肯定是**Dog或者其父类、间接父类**,根据里氏替换原则,任何使用父类的地方可以被它的子类替换,所以我们可以向 list 中添加**Dog或其子类、间接子类**.但是当我们从 list 中取数据的时候,由于不知道 list 中存放的具体是什么类型,在 java 中 Object 是所有类型的父类,所以这里取到的数据返回的**Object**类型.

一般情况下,我们获取到Object可以通过`className`或者`instanceof`来判断具体类型,但我们就先忽略吧.
像这种只写入而不读取的泛型类型声明情况称之为**消费者 Consumer**.

#### 无边界通配符

还有一种无边界通配符,用单问号表示：List<?>,也就是没有任何限定,相当于`? extends Object`.需要注意的是,它和不使用类型的 List 还是有区别的：
* List<?> list表示的是列表保存某个特定类型的对象,但我们不能向其中添加任何元素,因为我们不清楚 list 中保存的是那种类型
* 没有泛型参数的 List 表示该列表持有的元素类型是 Object,因此可以添加任何类型的对象,但编译器会有警告信息.

#### 小结
小小的总结一下：
利用`? extends`形式的通配符可以实现泛型的向上转型,也就是支持协变.但使用上通配符后编译器为了保证运行时的安全,会限定对其写的操作,开放读的操作也就是**只能读取不能修改**
利用`? super T`形式的通配符可以实现泛型的向下转型,也就是支持逆变,与上通配符相反,下边界通配符通常限定读的操作,开放写的操作,也就是**只能修改不能读取**

Joshua Bloch 在其著作《Effective Java》第三版 中很好地解释了该问题 (第 31 条：“利用有限制通配符来提升 API 的灵活性”). 他称那些你只能从中读取的对象为生产者, 并称那些只能向其写入的对象为消费者.他建议：
>为了灵活性最大化,在表示生产者或消费者的输入参数上使用通配符类型.

他还提出了以下助记符：PECS 代表生产者-Extends、消费者-Super(Producer-Extends, Consumer-Super).

### kotlin 中的泛型通配符
理清楚了 java 中的泛型通配符,接着我们看一下 kotlin 中的通配符,相对于 Java 的通配符提出了一种新的定义：**声明处型变(declaration-site variance)**与**类型投影(type projections)**
先从 kotlin 中的通配符说起：
和 java 泛型一样，kotlin 中的泛型也是不变的，同样的，也提供了相应的关键字来支持**协变**和**逆变**
* 使用关键字`out`来支持协变，等同于 Java 中的上界通配符`? extends`。
* 使用关键字`in`来支持逆变，等同于 Java 中的下界通配符`? super`。

``` kotlin
val outList: MutableList<out TestMain.Animal> = mutableListOf()
val outListItem: TestMain.Animal = outList[0]


val inList: MutableList<in TestMain.Animal> = mutableListOf()
inList.add(TestMain.Dog())
inList.add(TestMain.Bird())
inList.add(TestMain.Poodle())
val inListItem: Any? = inList[0]
```
无非是换了个写法而已，没多大差别.不过需要注意一下，kotlin 同时支持使用处型变和声明处型变。
举一个用烂了的例子
``` kotlin
class Producer<out T> {
    fun produce(): T {
        return null as T
    }
}
class Consumer<in T> {
    fun consume(t: T) {
        println(t)
    }
}
val producer: Producer<TestMain.Animal> = Producer()
val animal: TestMain.Animal = producer.produce()

val consumer: Consumer<TestMain.Animal> = Consumer()
consumer.consume(TestMain.Dog())
```
如果我们确认泛型参数只用来输入或者输出，可以在声明处直接添加`in`或者`out`.当然也可以在使用处添加声明
``` kotlin
class Producer1<T> {
    fun produce(): T? {
        return null
    }
}
class Consumer1<T> {
    fun consume(t: T) {
        println(t)
    }
}
val producer1: Producer1<out TestMain.Animal> = Producer1()
val animal1: TestMain.Animal? = producer1.produce()

val consumer1: Consumer1<in TestMain.Animal> = Consumer1()
consumer1.consume(TestMain.Dog())
```
这里也就是经常说的 **消费者 in, 生产者 out**

#### 类型投影

这个东西可以理解为就是一个概念，根据官方描述是这样的:
将类型参数`T`声明为`out`非常简单，并且能避免使用处子类型化的麻烦，但是有些类实际上不能限制为只返回`T`！一个很好的例子是`Array`：
``` kotlin
class Array<T>(val size: Int) {
    operator fun get(index: Int): T { …… }
    operator fun set(index: Int, value: T) { …… }
}
```
该类在`T`上既不能是协变的也不能是逆变的。这造成了一些不灵活性。考虑下述函数：
``` kotlin
fun copy(from: Array<Any>, to: Array<Any>) {
    assert(from.size == to.size)
    for (i in from.indices)
        to[i] = from[i]
}
```
这个函数应该将项目从一个数组复制到另一个数组。让我们尝试在实践中应用它：
``` kotlin
val ints: Array<Int> = arrayOf(1, 2, 3)
val any = Array<Any>(3) { "" } 
copy(ints, any)
//   ^ 其类型为 Array<Int> 但此处期望 Array<Any>
```
这里我们遇到同样熟悉的问题：`Array<T>`在`T`上是**不型变**的，因此`Array<Int>` 与 `Array<Any>` 都不是另一个的子类型。为什么？ 再次重复，因为`copy`可能有非预期行为，例如它可能尝试写一个`String`到`from`，并且如果我们实际上传递一个`Int`的数组，以后会抛`ClassCastException`异常。
如果需要禁止`copy`功能写入`from`，可以执行以下操作:
``` kotlin
fun copy(from: Array<out Any>, to: Array<Any>) { …… }
```
这就是类型投影：意味着`from`不仅仅是一个数组，而是一个`受限制`的**（投影的）**数组。 只可以调用返回类型为类型参数`T`的方法，如上，这意味着只能调用`get()`。 这就是使用处型变的用法，并且是对应于 Java 的 `Array<? extends Object>`, 但更简单。

你也可以使用`in`投影一个类型：
``` kotlin
fun fill(dest: Array<in String>, value: String) { …… }
```
`Array<in String>` 对应于 Java 的`Array<? super String>`，也就是说，你可以传递一个`CharSequence`数组或一个`Object`数组给`fill()`函数。
**以上信息来自 [kotlin 中文网]((https://book.kotlincn.net/text/generics.html))**
#### 星投影

有时你想说，你对类型参数一无所知，但仍然希望以安全的方式使用它。 这里的安全方式是定义泛型类型的这种投影，该泛型类型的每个具体实例化都会是该投影的子类型。

Kotlin 为此提供了所谓的**星投影**语法：

* 对于`Foo <out T : TUpper>`，其中`T`是一个具有上界`TUpper`的协变类型参数，`Foo <*>`等价于`Foo <out TUpper>`。 意味着当`T`未知时，你可以安全地从`Foo <*>`读取`TUpper`的值。
* 对于`Foo <in T>`，其中`T`是一个逆变类型参数，`Foo <*>`等价于`Foo <in Nothing>`。 意味着当`T`未知时， 没有什么可以以安全的方式写入`Foo <*>`。
* 对于`Foo <T : TUpper>`，其中`T`是一个具有上界`TUpper`的不型变类型参数,`Foo<*>`对于读取值时等价于`Foo<out TUpper>` 而对于写值时等价于`Foo<in Nothing>`。

如果泛型类型具有多个类型参数，则每个类型参数都可以单独投影。 例如，如果类型被声明为 interface Function <in T, out U>，可以使用以下星投影：

* `Function<*, String>` 表示`Function<in Nothing, String>`。
* `Function<Int, *>` 表示`Function<Int, out Any?>`。
* `Function<*, *>` 表示`Function<in Nothing, out Any?>`。

**以上信息来自 [kotlin 中文网]((https://book.kotlincn.net/text/generics.html))**

#### 泛型方法

不仅类可以有类型参数。函数也可以有。类型参数要放在函数名称之前：
``` kotlin
fun <T> singletonList(item: T): List<T> {
    // ……
}

fun <T> T.basicToString(): String { // 扩展函数
    // ……
}
```
要调用泛型函数，在调用处函数名之后指定类型参数即可：
``` kotlin
val l = singletonList<Int>(1)
```

可以省略能够从上下文中推断出来的类型参数，所以以下示例同样适用：
``` kotlin
val l = singletonList(1)
```

#### 泛型约束

能够替换给定类型参数的所有可能类型的集合可以由**泛型约束**限制。
最常见的约束类型是上界，与Java的`extends`关键字对应：
``` kotlin
fun <T : Comparable<T>> sort(list: List<T>) {  …… }
```
冒号之后指定的类型是上界，表明只有`Comparable<T>`的子类型可以替代`T`。 例如：
``` kotlin
sort(listOf(1, 2, 3)) // OK。Int 是 Comparable<Int> 的子类型
sort(listOf(HashMap<Int, String>())) // 错误：HashMap<Int, String> 不是 Comparable<HashMap<Int, String>> 的子类型
```
默认的上界（如果没有声明）是 Any?。在尖括号中只能指定一个上界。 如果同一类型参数需要多个上界，需要一个单独的 where-子句：
``` kotlin
fun <T> copyWhenGreater(list: List<T>, threshold: T): List<String>
    where T : CharSequence,
          T : Comparable<T> {
    return list.filter { it > threshold }.map { it.toString() }
}
```
所传递的类型必须同时满足`where`子句的所有条件。在上述示例中，类型`T`必须**既实现了 CharSequence 也实现了 Comparable**。
这里需要注意的是，where 子句后面的第一个类型可以是接口也可以是抽象类、实现类，后续的类型只能是接口。在 Java 中也一样
``` Java
public interface MyInterface {
    void test();
}
public abstract class MyAbstractClass {
    public abstract void test();
}

public <T extends MyInterface & MyAbstractClass> void test(T t){ //errpr
    t.test();
}
public <T extends  MyAbstractClass & MyInterface > void test(T t){
    t.test();
}
    
```
原因就是 java 中不可以多继承但可以多实现

####  @UnsafeVariance
差点忘了这东西，这个注解就是告诉编译器我知道我在做什么，并且保证不会出问题，忽略协变和逆变的约束就好了
比如 kotlin 中的`Collection`这个类中的`contains`、`containsAll`方法
``` kotlin
public interface Collection<out E> : Iterable<E> {
    public operator fun contains(element: @UnsafeVariance E): Boolean
    public fun containsAll(elements: Collection<@UnsafeVariance E>): Boolean
}
```
> 对于协变的类型，通常我们是不允许将泛型类型作为传入参数的类型的，或者说，对于协变类型，我们通常是不允许其涉及泛型参数的部分被改变的。
> 这也很容易解>释为什么 MutableCollection 是不变的，而 Collection 是协变的，因为在 Kotlin 当中，前者是可被修改的，后者是不可被修改的。
> 逆变的情形正好相反，即不可以将泛型参数作为方法的返回值。

比如这种情形，为了让编译器放过一马，我们就可以用 @UnsafeVariance 来告诉编译器：“我知道我在干啥，保证不会出错，你不用担心”。
**以上信息来自[深入解析Kotlin 泛型](https://zhuanlan.zhihu.com/p/143380842)**

#### reified 关键字
由于存在类型擦除，导致我们无法在运行时获取泛型的具体类型，有些操作无法实现，比如
``` Java
public static <T> void testOne(Object param){
    if(param instanceof T){
        System.out.println("T");
    }
}
```
当然，在 kotlin 中也不行。
但在 java 中我们通常会传入一个`Class<T>`来做相应的操作，在 kotlin 中同样也可以，不过 kotlin 中有一个更简单的方法:使用`reified`配合`inline`来实现
``` kotlin
inline fun <reified T> printIfTypeMatch(item: Any) {
    if (item is T) { // 👈 这里就不会在提示错误了
        println(item)
    }
}
```
我们经常用的 gson解析数据、反序列化的时候经常遇到
``` Java
public <T> T fromJson(String json, Class<T> classOfT) throws JsonSyntaxException { 
    
} 
```
这里就是通过多传入一个`Class<T>`来解决这个问题，在 kotlin 中我们可以通过扩展来变化一下
``` kotlin
inline fun <reified T> Gson.fromJson(json: String): T{ 
     return fromJson(json, T::class.java) 
 } 
```
我们给 Gson 添加了一个扩展方法，在这个方法中，通过`inline`和`reified`关键字将泛型`T`变成了一个真实可用的类型，这两个关键字缺一不可。这里就简单的认为内联方法(inline)是将方法在编译时复制到调用处，使得泛型 T 的类型在编译时就可以确定。当然这么理解不是特别正确。后面学到`inline`、`noinline`、`crossinline`这几个关键字的时候再说吧


参考：

[泛型：in、out、where](https://book.kotlincn.net/text/generics.html)
[Generics: in, out, where](https://kotlinlang.org/docs/generics.html)
[深入理解Java和Kotlin中的泛型](https://ethanhua.github.io/2018/01/09/genericity/)
[深入解析Kotlin 泛型](https://zhuanlan.zhihu.com/p/143380842)

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

未学习：
* 关键字
  * object
  * Unit
  * Nothing
  * inline,noinline,crossinline

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



