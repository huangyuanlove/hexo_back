---
title: gradle任务
tags: [gradle,Android]
date: 2018-11-19 10:56:26
keywords: gradle基础,Android构建脚本,gradle任务,gradle task
---

参考《Android Gradle 权威指南》第四章Gradle任务，主要介绍任务的创建方式、访问任务、任务分组和描述、<<操作符、任务的执行分析、任务排序、启用和禁用、断言、规则等。

<!--more-->

#### 任务的创建方式

##### 直接以任务名的方式创建

``` groovy
def Task temp = task(createTaskByName)
temp.doLast {
    println "原型为 Task task(String name) throws InvalidUserDataException"
}
```

上面代码中的`createTaskByName`就是我们创建的任务，可以通过`gradle -q createTaskByName`来执行这个任务。这种创建方式本质就上就是调用`Project`的`task(String name)`方法来创建任务，然后将`Task`类型的返回值赋值给我们定义的temp变量，最后通过`doLast`方式来配置任务。

##### 任务名字+任务配置的Map对象

``` groovy
def Task temp = task(createTaskByName,group: 'custom')
temp.doLast {
    println "原型为 Task task(String name) throws InvalidUserDataException"
}
```

和第一种方式大同小异，只是多了一个Map参数用以对task进行配置，比如上面的代码指定了该task所属的group，当我们在执行 `gradle -q tasks --all`就可以在对应的group中找到该任务。下面列出的是可用配置：

| 配置项 | 描述 | 默认值 |
| :------: | :----: | :------: |
| type | 基于一个存在的Task来创建，类似继承 | DefaultTask |
| overwirte | 是否替换存在的Task，可以和type配合使用 | false |
| dependsOn | 用于配置任务依赖 | [] |
| action | 添加到任务中的一个action或者一个闭包 | null |
| description | 任务描述 | null |
| group | 任务分组 | null |

##### 任务名字+闭包形式

``` groovy
task temp {
    description "这是任务名字+闭包形式创建任务"
    doLast {
    	println "通过任务名字+闭包形式创建任务"
    	println "${description}"
    }
}
```

因为Map的方式所能配置的信息有限，所以常见的创建任务是已这种方式进行，或者调用tasks.create方法：

``` groovy
tasks.create('temp'){
description "这是任务名字+闭包形式创建任务"
    doLast {
    println "通过任务名字+闭包形式创建任务"
    println "${description}"
    }
}
```

#### 访问任务

1. 通过任务名字访问

``` groovy
task temp{}
//第一种方式
temp.doLast{}
//第二种方式
tasks['temp'].doLast{}
```

访问的时候，任务名就是key,这里说法有点不恰当，因为tasks并不是一个map，`[]`在groovy中是一个操作符，在groovy中的操作符都有对应的方法让我们重载，`a[b]`对应的是`a.getAt(b)`这个方法，对应上面的代码`tasks['temp']`其实是调用的`tasks.getAt('temp')`这个方法，在`gradle`源码中，发现调用的是`findByName(String name)`实现的。

然后就是通过路径访问，有两种方式，一种是`get`，一种是`find`，区别在于使用`get`的时候如果找不到该任务就是抛出`UnknownTaskException`，而find则会返回`null`.

``` groovy
task temp
tasks['temp'].doLast{
    println tasks.findByPath('temp')
    println tasks.getByPath('temp')
}
```

最后就是通过名称访问，方式、区别和上面通过路径访问是一样的。

``` groovy
task temp
tasks['temp'].doLast{
    println tasks.findByName('temp')
    println tasks.getByName('temp')
}
```
需要注意的是，通过路径访问的时候，参数值可以是路径值也可以是任务名字,但是通过名字访问的时候，参数值只能是任务的名字，不能为路径。

####  <<操作符

`<<`操作符在Gradle的Task上是doLast方法的短标记形式。

`<<`是操作符，在Groovy中是可以重载的，`a<<b`对应的是`a.leftShift(b)`方法，所以Task接口中肯定有同一个leftShift方法重载的`<<`，在Gradle源码中

``` java
Task.java
/**
     * <p>Adds the given closure to the end of this task's action list.  The closure is passed this task as a parameter
     * when executed. You can call this method from your build script using the &lt;&lt; left shift operator.</p>
     *
     * @param action The action closure to execute.
     * @return This task.
     *
     * @deprecated Use {@link #doLast(Closure action)}
     */
    @Deprecated
    Task leftShift(Closure action);
```

leftShift方法和doLast方法实现

``` java
 @Override
    public Task leftShift(final Closure action) {
        DeprecationLogger.nagUserOfDiscontinuedMethod("Task.leftShift(Closure)", "Please use Task.doLast(Action) instead.");

        hasCustomActions = true;
        if (action == null) {
            throw new InvalidUserDataException("Action must not be null!");
        }
        taskMutator.mutate("Task.leftShift(Closure)", new Runnable() {
            public void run() {
                getTaskActions().add(taskMutator.leftShift(convertClosureToAction(action, "doLast {} action")));
            }
        });
        return this;
    }
```

``` java
@Override
    public Task doLast(final Closure action) {
        hasCustomActions = true;
        if (action == null) {
            throw new InvalidUserDataException("Action must not be null!");
        }
        taskMutator.mutate("Task.doLast(Closure)", new Runnable() {
            public void run() {
                getTaskActions().add(convertClosureToAction(action, "doLast {} action"));
            }
        });
        return this;
    }
```

#### 任务执行分析

当我们在执行一个Task的时候，其实就是执行其拥有的actions列表，这个列表保存在Task对象实例中的actions成员变量中，在`AbstractTask`类中有这么一个方法：

``` java
 @Override
    public List<ContextAwareTaskAction> getTaskActions() {
        if (actions == null) {
            actions = new ArrayList<ContextAwareTaskAction>(3);
        }
        return actions;
    }
```

我们写个demo来看一下actions怎么工作的

``` groovy
Task myTask = task test(type : CustomTask)

myTask.doFirst {
    println "doFirst"
}

myTask.doLast {
    println "doLast"
}

class CustomTask extends DefaultTask{

    @TaskAction
    def doSelf(){
        println "执行doSelf"
    }
}

```

上面代码定义了一个Task类型的CustomTask，并且声明了一个被`TaskAction`注解标准的`doSelf`方法。意思是该方法就是Task本身执行的方法，执行`test`任务，得到输出

> doFirst
> 执行doSelf
> doLast

我们可以看一下具体源码怎么写的

当我们使用Task方法创建一个task的时候，Gradle会解析其带有TaskAction标注的方法作为其Task执行的Action，然后通过

``` java
 @Override
    public void prependParallelSafeAction(final Action<? super Task> action) {
        if (action == null) {
            throw new InvalidUserDataException("Action must not be null!");
        }
        getTaskActions().add(0, wrap(action));
    }
```

添加到actions list里面。这时候Task刚被创建，所以不会有其他的action。

``` java
@Override
    public Task doFirst(final Action<? super Task> action) {
        return doFirst("doFirst {} action", action);
    }

    @Override
    public Task doFirst(final String actionName, final Action<? super Task> action) {
        hasCustomActions = true;
        if (action == null) {
            throw new InvalidUserDataException("Action must not be null!");
        }
        taskMutator.mutate("Task.doFirst(Action)", new Runnable() {
            public void run() {
                getTaskActions().add(0, wrap(action, actionName));
            }
        });
        return this;
    }
```

``` java
@Override
    public Task doLast(final Action<? super Task> action) {
        return doLast("doLast {} action", action);
    }

    @Override
    public Task doLast(final String actionName, final Action<? super Task> action) {
        hasCustomActions = true;
        if (action == null) {
            throw new InvalidUserDataException("Action must not be null!");
        }
        taskMutator.mutate("Task.doLast(Action)", new Runnable() {
            public void run() {
                getTaskActions().add(wrap(action, actionName));
            }
        });
        return this;
    }
```

可以看到，doFirst每次都添加到list的最前面，doLast每次都添加到最后面。最后这个actions list就按照顺序形成了doFirst、doSelf、doLast三部分的Actions。

#### 任务排序

这个并没有真正实现排序功能，而是通过`shouldRunAfter`和`mustRunAfter`这两个方法，他们可以控制一个任务应该或者一定在某个任务之后执行。

`taskB.mustRunAfter(taskA)`表示`taskB`必须在`taskA`执行之后执行，这个规则比较严格

#### 任务的启用和禁用

Task中有个`enable`属性，用于启用和禁用任务，默认是`true`，表示启用，设置为`false`，则禁止该任务执行，输出会提示该任务被跳过。

#### 任务的onlyIf断言

断言就是一个条件表达式，Task有一个`onlyIf`方法，它接受一个闭包作为参数，如果该闭包返回true则执行该任务，否则跳过。

假如我们首发渠道是应用宝，直接build会编译出来所有包，现在我们就采用onlyIf的方式通过属性来控制：

``` groovy
final String BUILD_APP_ALL = "all";
final String BUILD_APP_SHOUFA = "shoufa";
final String BUILD_APP_EXCLUDE_SHOUFA="exclude_shoufa";

task qqRelease <<{
    println "应用宝"
}

task baiduRelease <<{
    println "百度"
}

task huaweiRelease <<{
    println "华为"
}

task miuiRelease <<{
    println "小米"
}

task test <<{
    group BasePlugin.BUILD_GROUP
    description "打渠道包"
}

test.dependsOn qqRelease,baiduRelease,huaweiRelease,miuiRelease

qqRelease.onlyIf {
    def execute = false;

    if(project.hasProperty("build_apps")){
        Object buildApp = project.property("build_apps")
        if(BUILD_APP_SHOUFA.equals(buildApp) || BUILD_APP_ALL.equals(buildApp)){
            execute = true
        }else{
            execute = false
        }
    }else{
        execute = true
    }
    execute
}

baiduRelease.onlyIf {
    def execute = false;
    
    if(project.hasProperty("build_apps")){

        String buildApp = project.property("build_apps")
        if(BUILD_APP_SHOUFA.equals(buildApp) || BUILD_APP_ALL.equals(buildApp)){
            execute = true
            
        }else{
            execute = false
            
        }
    }else{
        execute = true
        
    }
    execute

}

huaweiRelease.onlyIf {
    def execute = false;

    if(project.hasProperty("build_apps")){
        Object buildApp = project.property("build_apps")
        if(BUILD_APP_EXCLUDE_SHOUFA.equals(buildApp) || BUILD_APP_ALL.equals(buildApp)){
            execute = true
        }else{
            execute = false
        }
    }else{
        execute = true
    }
    execute
}

miuiRelease.onlyIf {
    def execute = false;

    if(project.hasProperty("build_apps")){
        Object buildApp = project.property("build_apps")
        if(BUILD_APP_EXCLUDE_SHOUFA.equals(buildApp) || BUILD_APP_ALL.equals(buildApp)){
            execute = true
        }else{
            execute = false
        }
    }else{
        execute = true
    }
    execute
}
```

上面定义了4个渠道，其中百度和应用宝是首发，通过build_apps属性来控制要打哪些渠道包

>//打所有渠道包
>
>gradle test
>
>gradle -P build_apps=all test
>
>//打首发包
>
>gradle -P build_apps=shoufa test
>
>//打非首发包
>
>gradle -P build_apps=exclude_shoufa test

命令行中的-P意思是为Project执行K-V格式属性的键值对

#### 任务规则

我们知道创建的人物都爱TaskContainer里，是由其进行管理的，所以当我们访问任务的时候都是通过TaskContainer进行访问，而TaskContainer又是一个NamedDomainObjectCollection，所以我们说的任务规则其实就是NamedDomainObjectCollection的规则。

NamedDomainObjectCollection是一个具有唯一不变名字的域对象的集合，它里面所有的元素都有一个唯一不变的名字，该名字是String类型，所以我们可以通过名字获取该元素。但是这个唯一的名字可能不存在，具体到任务中就是说你想获取的这个任务不存在，这时候就会调用我们添加的规则来处理这种异常情况。

``` java	
public T findName(String name){
    T value = findByNameWithoutRules(name);
    if(value != null){
        return value;
    }
    applyRules(name);
    return findByNameWithoutRules(name);
}
```

以名字查找的时候，如果没有找到则调用applyRules(name)应用我们添加的规则。

我们可以通过调用addRule来添加我们自定义的规则，它有两个用法:

``` java
 /**
     * Adds a rule to this collection. The given rule is invoked when an unknown object is requested by name.
     *
     * @param rule The rule to add.
     * @return The added rule.
     */
    Rule addRule(Rule rule);

    /**
     * Adds a rule to this collection. The given closure is executed when an unknown object is requested by name. The
     * requested name is passed to the closure as a parameter.
     *
     * @param description The description of the rule.
     * @param ruleAction The closure to execute to apply the rule.
     * @return The added rule.
     */
    Rule addRule(String description, Closure ruleAction);
```

一个是直接添加一个Rule，另一个是通过闭包配置成一个Rule再添加，两种方式大同小异。

当我们执行、依赖一个不存在的任务时，Gradle会执行失败，失败信息是任务不存在。我们使用规则对其进行改造，不会执行失败，而是打印提示信息：

``` groovy
tasks.addRule("对该规则的一个描述"){
    String taskName ->
        task(taskName) <<{
            println("该${taskName}任务不存在，请查证后再执行")
        }
}

task testA {
    dependsOn missTask
}
```

