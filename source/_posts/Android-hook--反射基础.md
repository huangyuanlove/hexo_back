---
title: Android hook--反射基础
date: 2018-08-06 15:18:09
tags: [Android,Java]
keywords: AndroidHook,反射,动态代理

---
假如你已经非常熟悉java中反射(reflect)和代理(Proxy)，那你还在这里看我这篇文章纯粹就是浪费时间了。
<!--more-->
#### 反射是什么
官方介绍
> Reflection is commonly used by programs which require the ability to examine or modify the runtime behavior of applications running in the Java virtual machine. This is a relatively advanced feature and should be used only by developers who have a strong grasp of the fundamentals of the language. With that caveat in mind, reflection is a powerful technique and can enable applications to perform operations which would otherwise be impossible.

概括来讲就是：** 反射这个功能很XX **
大家常见的对反射机制的概念:
在Java中的反射机制是指在运行状态中，对于任意一个类都能够知道这个类所有的属性和方法；并且对于任意一个对象，都能够调用它的任意一个方法；这种动态获取信息以及动态调用对象方法的功能成为Java语言的反射机制。

#### 涉及到的类

* `Class`：反射的核心类，可以获取类的属性，方法等信息。 
* `Field`：Java.lang.reflec包中的类，表示类的成员变量，可以用来获取和设置类之中的属性值。 
* `Method`： Java.lang.reflec包中的类，表示类的方法，它可以用来获取类中的方法信息或者执行方法。 
* `Constructor`： Java.lang.reflec包中的类，表示类的构造方法。
  
#### 简介

先写一个简单的Person类当做目标类
``` java
class Person {
	public String name;
	private String nickName;
	int age;

	public Person(){}
	protected  Person(int age) {
		this. age = age;
	}
	
	private Person(String name) {
		this.name = name;
	}
	
	public Person(String name,String nickName) {
		this.name = name;
		this.nickName = nickName;
	}
	
	public Person(String name, String nickName, int age) {
		super();
		this.name = name;
		this.nickName = nickName;
		this.age = age;
	}

	@Override
	public String toString() {
		return "Person [name=" + name + ", nickName=" + nickName + ", age=" + age + "]";
	}

}
```

##### 获取想要操作的类的Class对象
1. Object.getClass();
  ``` java
    Person person = new Person();
		Class personClass = person.getClass();
  ```
2. 任何数据类型（包括基本数据类型）都有一个“静态”的class属性
  ``` java
    Class personClass2 = Person.class;
  ```
3. 通过Class类的静态方法：forName（String  className）(常用)   
  ``` java
    Class personClass3 = Class.forName("com.huangyuanlove.Person");
  ```
  需要注意的是，在运行期间，一个只有一个Class对象：
  ``` java
		System.out.println(personClass);
		System.out.println(personClass == personClass2);
		System.out.println(personClass == personClass3);
  ```
  输出：
  > class com.huangyuanlove.Person
    true
    true

##### 调用Class类中的方法
###### 获取构造方法并创造对象：
   ``` java
   	Person person = new Person();
		Class personClass = person.getClass();

		// 获取所有的公有构造方法
		Constructor[] publicConstructors = personClass.getConstructors();
		System.out.println("获取所有的公有构造方法");
		for (Constructor c : publicConstructors) {
			System.out.println(c);
		}

		// 获取所有的构造方法
		Constructor[] allConstructors = personClass.getDeclaredConstructors();
		System.out.println("获取所有的构造方法");
		for (Constructor c : allConstructors) {
			System.out.println(c);
		}

		// 获取公有，无参构造方法
		Constructor publicConstructorWithoutArgs = personClass.getConstructor();
		System.out.println("获取公有，无参构造方法");
		System.out.println(publicConstructorWithoutArgs);
		System.out.println(publicConstructorWithoutArgs.newInstance());

		// 获取私有，有一个String类型参数的构造方法
		Constructor publicConstructorWithOneStringArgs = personClass.getDeclaredConstructor(String.class);
		System.out.println("获取私有，有一个String类型参数的构造方法");
		System.out.println(publicConstructorWithOneStringArgs);
		personClass.getDeclaredField("nickName").setAccessible(true);
		System.out.println(publicConstructorWithOneStringArgs.newInstance("xuan"));

		// 获取公有，有两个个String类型参数的构造方法
		Constructor publicConstructorWithTwoStringArgs = personClass.getConstructor(String.class, String.class);
		System.out.println("获取公有，有两个个String类型参数的构造方法");
		System.out.println(publicConstructorWithTwoStringArgs);
		publicConstructorWithTwoStringArgs.setAccessible(true);
		System.out.println(publicConstructorWithTwoStringArgs.newInstance("xuan", "huangyuan"));
   ```
  这里需要注意的是，在获取私有，有一个String类型参数的构造方法，并调用`newInstance`方法的时候会抛出异常，这是因为该构造方法中的`nickName`字段是私有的,将其注释掉可获得如下输出
  输出：
  > 获取所有的公有构造方法
    public com.huangyuanlove.Person(java.lang.String,java.lang.String,int)
    public com.huangyuanlove.Person(java.lang.String,java.lang.String)
    public com.huangyuanlove.Person()
    获取所有的构造方法
    public com.huangyuanlove.Person(java.lang.String,java.lang.String,int)
    public com.huangyuanlove.Person(java.lang.String,java.lang.String)
    private com.huangyuanlove.Person(java.lang.String)
    protected com.huangyuanlove.Person(int)
    public com.huangyuanlove.Person()
    获取公有，无参构造方法
    public com.huangyuanlove.Person()
    Person [name=null, nickName=null, age=0]
    获取私有，有一个String类型参数的构造方法
    private com.huangyuanlove.Person(java.lang.String)
    获取公有，有两个个String类型参数的构造方法
    public com.huangyuanlove.Person(java.lang.String,java.lang.String)
    Person [name=xuan, nickName=huangyuan, age=0]

###### 获取成员变量并进行操作
  ``` java
    Person person = new Person();
		Class personClass = person.getClass();

		// 获取所有的公共成员变量
		Field publicFields[] = personClass.getFields();
		System.out.println("获取所有的公共成员变量");
		for (Field f : publicFields) {
			System.out.println(f);
		}
		// 获取所有的成员变量
		Field allFields[] = personClass.getDeclaredFields();
		System.out.println("获取所有的成员变量");
		for (Field f : allFields) {
			System.out.println(f);
		}

		// 获取某个公有成员变量并赋值
		Field nameField = personClass.getField("name");
		nameField.set(person, "huangyuan");
		System.out.println(person);

		// 获取某个私有成员变量并赋值
		Field nickNameField = personClass.getDeclaredField("nickName");
    //因为nickName是私有的，所有需要先设置可访问
		nickNameField.setAccessible(true);
		nickNameField.set(person, "xuan");
		System.out.println(person);
  ```
  可以得到如下输出：
  > 获取所有的公共成员变量
    public java.lang.String com.huangyuanlove.Person.name
    获取所有的成员变量
    public java.lang.String com.huangyuanlove.Person.name
    private java.lang.String com.huangyuanlove.Person.nickName
    int com.huangyuanlove.Person.age
    Person [name=huangyuan, nickName=null, age=0]
    Person [name=huangyuan, nickName=xuan, age=0]

  重要的事情来了，一定要记住，谁要是想上面那样反射获取类的公有成员变量然后进行赋值操作，肯定被骂的祸国殃民、民不聊生、生灵涂炭，都public了你还反射。
  获取私有变量的时候需要使用`getDeclaredField`方法，否则会抛出`noSuchFieldException`

###### 获取方法并进行调用
   在类中添加两个方法
   ``` java
    public void sayHi() {
        System.out.println("hi");
    }
    
    private void saySomeThing(String someThing) {
        System.out.println(someThing);
    }
   ```
  反射获取类方法并进行调用
  ``` java
    Person person = new Person();
		Class personClass = person.getClass();

		// 获取所有公共方法
		System.out.println("获取所有公共方法");
		Method publicMethods[] = personClass.getMethods();
		for (Method m : publicMethods) {
			System.out.println(m.getName());
		}

		// 获取所有方法
		System.out.println("获取所有方法");
		Method allMethods[] = personClass.getDeclaredMethods();
		for (Method m : allMethods) {
			System.out.println(m.getName());
		}

		// 获取指定的公有方法
		System.out.println("获取指定的公有方法");
		Method publicMethodWithoutArgs = personClass.getMethod("sayHi");
		publicMethodWithoutArgs.invoke(person);

		// 获取指定的私有方法
		System.out.println("获取指定的私有方法");
		Method privateMethodWithStringArgs = personClass.getDeclaredMethod("saySomeThing", String.class);
		privateMethodWithStringArgs.setAccessible(true);
		privateMethodWithStringArgs.invoke(person, "someThing");
  ```
  得到输出：
  > 获取所有公共方法
    toString
    sayHi
    wait
    wait
    wait
    equals
    hashCode
    getClass
    notify
    notifyAll
    获取所有方法
    toString
    sayHi
    saySomeThing
    获取指定的公有方法
    hi
    获取指定的私有方法
    someThing

  获取到的方法中并不包含构造方法，但是包含从父类继承下来的公共方法。和上面的获取成员变量赋值一样，谁要是反射去获取公共方法再去调用，基本上就凉了。

#### 静态代理和动态代理
##### 静态代理
简单来说，代理就是用一个代理类来封装一个委托类，这样做有两个好处：可以隐藏委托类的具体实现；可以在不改变委托类的情况下增加额外的操作。而静态代理，就是在程序运行之前，代理类就已经存在了。静态代理一般的实现方式为：委托类和代理类都实现同一个接口或者是继承自同一个父类，然后在代理类中保存一个委托类的对象引用（父类或者父类接口的对象引用），通过给构造器传入委托类的对象进行初始化，在同名方法中通过调用委托类的方法实现静态代理。除此之外，在代理类同名方法中还可以实现一些额外的功能。代码如下：
RealObject类为委托类，SimpleProxy类为代理类：
``` java
interface Interface {
    void doSomething();

    void somethingElse(String arg);
}

class RealObject implements Interface {
    @Override
    public void doSomething() {
        // TODO Auto-generated method stub
        System.out.println("doSomething");
    }

    @Override
    public void somethingElse(String arg) {
        // TODO Auto-generated method stub
        System.out.println("somethingElse " + arg);
    }
}

class SimpleProxy implements Interface {
    // 保存委托类（父接口的引用）
    private Interface proxied;

    // 传入委托类的对象用于初始化
    public SimpleProxy(Interface proxied) {
        this.proxied = proxied;
    }

    // 两个同名方法中还实现了其他的功能
    @Override
    public void doSomething() {
        // TODO Auto-generated method stub
        System.out.println("SimpleProxy doSomething");
        proxied.doSomething();
    }

    @Override
    public void somethingElse(String arg) {
        // TODO Auto-generated method stub
        System.out.println("SimpleProxy somethingElse " + arg);
        proxied.somethingElse(arg);
    }
}

public class SimpleProxyDemo {
    public static void main(String[] args) {
        consumer(new RealObject());
        consumer(new SimpleProxy(new RealObject()));
    }

    public static void consumer(Interface iface) {
        iface.doSomething();
        iface.somethingElse("bonobo");
    }
}
```
##### 动态代理
静态代理的局限性在于，代理类需要在程序运行之前就编写好，而动态代理则可以在程序运行的过程中动态创建并处理对所代理方法的调用。在动态代理中，需要定义一个中介类，这个类实现InvocationHandle接口（主要是里面的invoke方法）。这个中介类位于委托类和代理类之间，作为一个调用处理器而存在。它保存一个委托类的引用，通过传入委托类对象进行初始化；然后在invoke方法中，实现对委托类方法的调用，并增加需要的额外操作。在需要使用动态代理时，首先通过Proxy类中的newProxyInstance方法得到代理类对象（方法的三个参数分别是：（通常是委托类实现接口的）类加载器，希望代理类实现的接口列表（通常也是委托类实现的接口），以及一个调用处理器的对象），然后通过这个代理类对象直接调用代理类的方法。这种调用实际上会通过调用处理器调用invoke方法，进而实现对委托类相应方法的调用。

注意在动态代理中，只实现了一个调用处理器，而没有真正实现代理类。代理类对象是通过Proxy类中的newProxyInstance方法得到的。这样，不管你在调用委托类任何方法时需要加入的额外操作都可以仅仅在调用处理器中的invoke方法中实现就可以了。代码示例如下
``` java
public class SimpleDynamiProxyDemo {
    public static void consumer(Interface iface) {
        iface.doSomething();
        iface.somethingElse("bonobo");
    }

    public static void main(String[] args) {
        RealObject real = new RealObject();
        consumer(real);
        // 通过Proxy.newProxyInstance方法得到代理类对象
        Interface proxy = (Interface) Proxy.newProxyInstance(Interface.class.getClassLoader(),
                new Class[] { Interface.class }, new DynamicProxyHandler(real));
        // 通过代理类对象直接调用方法，会被重定向到调用处理器上的invoke方法
        consumer(proxy);

    }
}

// 中介类（调用处理器）
class DynamicProxyHandler implements InvocationHandler {
    // 保存一个委托类的对象
    private Object proxied;

    public DynamicProxyHandler(Object proxied) {
        this.proxied = proxied;
    }

    // 三个参数：代理类的引用，方法名和方法的参数列表
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        // TODO Auto-generated method stub
        System.out.println("**** proxy: " + proxy.getClass() + ", method: " + method + ", args: " + args);
        if (args != null) {
            for (Object arg : args) {
                System.out.println(" " + arg);
            }
        }
        // 实现对委托类方法的调用，参数表示委托类对象和参数
        return method.invoke(proxied, args);
    }
}
```
----
以上
以上静态代理和动态代理相关的文字代码出自于 https://www.cnblogs.com/hrcnblogs/p/8711418.html
