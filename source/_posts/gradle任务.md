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

