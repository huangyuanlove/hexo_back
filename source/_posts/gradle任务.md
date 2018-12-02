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

