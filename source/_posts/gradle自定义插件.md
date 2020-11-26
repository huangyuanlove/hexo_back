---
title: gradle自定义插件
date: 2020-11-26 07:06:31
tags: [gradle,Android]
keywords: gradle基础,gradle plugin,gradle插件
---
[groovy 语法入门](https://blog.huangyuanlove.com/2018/11/09/groovy%E8%AF%AD%E6%B3%95%E5%85%A5%E9%97%A8/)
[gradle 基础](https://blog.huangyuanlove.com/2018/11/11/gradle%E5%9F%BA%E7%A1%80/)
[gradle 任务](https://blog.huangyuanlove.com/2018/11/19/gradle%E4%BB%BB%E5%8A%A1/)
[gradle 插件](https://blog.huangyuanlove.com/2018/12/09/gradle%E6%8F%92%E4%BB%B6/)
[gradle-java 插件](https://blog.huangyuanlove.com/2018/12/19/gradle-java%E6%8F%92%E4%BB%B6/)
[gradle-android 插件](https://blog.huangyuanlove.com/2018/12/22/gradle-android%E6%8F%92%E4%BB%B6/)

前面简单的写了点关于gradle的以及gradle插件的东西,现在我们来看一下如何自定义插件,**本篇文章是基于AndroidStudio、Android工程进行讲述**。
<!--more-->
#### 存放插件源码
我们可以在以下几个地方存放我们的插件源码
##### Build Script
每个module中都会有build.gradle文件，我们可以在该文件中编写一些所需要的插件功能，好处是可以被自动编译并且包含在构建脚本的class path中(项目根目录下的build.gradle中buildScript中使用classPath依赖的插件)，坏处是不能被其他模块访问，插件功能没办法重用。
##### `buildSrc` Project
根据所选语言的不同，我们可以把插件代码放在`rootProjectDir/buildSrc/src/main/java`、`rootProjectDir/buildSrc/src/main/groovy`、`rootProjectDir/buildSrc/src/main/kotlin` 文件夹下，同样的，我们也不需要做额外的操作就可以在其他module中使用，但是不能在其他项目中引用
##### Standalone Project
我们可以为插件单独创建一个项目或者一个module，将它编译为jar包或者其他形式发布出去，使得其他项目可以引用


#### 编写插件代码
先看下写在`Build Script`中的构建脚本。
这里的`Build Script`指的是每个module都会有`build.gradle`文件，我们对每个module的某些编译配置选项也会在这里配置.
**首先需要明确的是，我们可以在`build.gradle`文件编写Groovy、Java代码 **，还可以回顾一下之前写的一坨文章看一下。
**以下代码我是在resource1模块的build.gradle文件中编写 **
先声明一个继承自`org.gradle.api.Plugin.Plugin`的类
``` groovy
class MyPlugin implements Plugin<Project> {
    @Override
    void apply(Project project) {
        project.task("greeting") {
            doLast {
                println("hello ")
            }
        }
    }
}
```
然后apply一下
``` groovy
apply plugin: MyPlugin
```
这时候点击`Sync Now`,会在对应的module中Tasks-->other分组中展示；双击该任务或者使用命令行可执行(./gradlew resource1:greeting);
如果想要像`apply plugin: 'com.android.library'`这种进行配置该如果办？我们可以这么做
``` groovy
class MyPlugin implements Plugin<Project> {
    @Override
    void apply(Project project) {
        def extension = project.extensions.create("myPlugin", MyPluginExtension)
        project.task("greeting") {
            doLast {
                println("$extension.greeter ,$extension.message")
            }
        }
    }
}

class MyPluginExtension {
    def message = "default message from MyPluginExtension"
    def greeter = "default greeter from MyPluginExtension"
}

apply plugin: MyPlugin
myPlugin {
    message = "hi"
    greeter = "xuan"
}
```
这里需要注意的是，`project.extensions.create`方法中传入的第一个参数是我们在`build.gradle`文件配置块的名字；
另外，如果我们想要在定义的类中使用一些三方的依赖包，