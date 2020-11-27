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
##### `buildSrc` Module
根据所选语言的不同，我们可以把插件代码放在`rootProjectDir/buildSrc/src/main/java`、`rootProjectDir/buildSrc/src/main/groovy`、`rootProjectDir/buildSrc/src/main/kotlin` 文件夹下，同样的，我们也不需要做额外的操作就可以在其他module中使用，但是不能在其他项目中引用
##### Standalone Project
我们可以为插件单独创建一个项目或者一个module，将它编译为jar包或者其他形式发布出去，使得其他项目可以引用


#### 编写插件代码
##### 先看下写在`Build Script`中的构建脚本。
这里的`Build Script`指的是每个module都会有`build.gradle`文件，我们对每个module的某些编译配置选项也会在这里配置.
**首先需要明确的是，我们可以在`build.gradle`文件编写Groovy、Java代码**，还可以回顾一下之前写的一坨文章看一下。
**以下代码我是在resource1模块的build.gradle文件中编写**
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
另外，我们需要注意一个作用域的问题：如果我们想要在定义的类中使用一些三方的依赖包，需要在**工程根目录下的build.gralde**文件中**buildscript**使用classpath添加依赖，比如我想使用`commons-lang3`中的StringUtil来判断字符串是否为空，则
``` groovy
buildscript {
    dependencies {
        classpath 'org.apache.commons:commons-lang3:3.11'
    }
}
```
然后在module中的build.gradle文件中使用，别忘了导入包。

##### 在`buildSrc` Module中
我们在工程中新建一个名字为`buildSrc`的文件夹，和各个module同级，然后按照module的格式，创建src/main/groovy|java|kotlin/package_name、build.gradle文件，将上面写的插件实现复制过来，
文件夹结构看起像这样
![gradle-plugin-buildSrc](/image/gradle/gradle-plugin/gradle-plugin-with-buildSrc.png)
然后在build.gradle文件中引入我们所需要的依赖、plugin等，看起来像这样
``` groovy
plugins {
    id 'java-gradle-plugin'
}
java {
    sourceCompatibility = JavaVersion.VERSION_1_7
    targetCompatibility = JavaVersion.VERSION_1_7
}
```
这里的pluginds相当于 `apply plugin:'java-gradle-plugin'`,这个插件是官方推荐使用的，相当于我们引用了`java`和`groovy`，并且添加了`gradleApi()`的依赖,可以看这里[https://docs.gradle.org/nightly/userguide/custom_plugins.html](https://docs.gradle.org/nightly/userguide/custom_plugins.html)。
我们还需要给我们的plugin取个名字，这里有两种方案
1. 官方现在推荐
在`build.gradle`中配置
``` groovy
gradlePlugin {
    plugins {
        simplePlugin {
            id = 'first-plugin'
            implementationClass = 'com.huangyuanlove.plugin.FirstPlugin'
        }
    }
}
```
2. 之前的写法
创建`main/resources/META-INF/gradle-plugins`文件夹，并在该文件夹下新建`first-plugin.properties`文件(这里的first-plugin就是插件的id)，在该文件中声明实现插件的类`implementation-class=com.huangyuanlove.plugin.FirstPlugin`

引用这个插件：
在使用这个插件的moudle中，
``` groovy
apply plugin: 'first-plugin'
greeting{
    message="hi"
    greeter="xuan"
}
```
然后同步一下就可以使用了

##### 作为一个独立模块

1. 编写、构建、发布
和上面差不多，新建一个`java library module`,然后像上面一样引入`java-gradle-plugin`,配置好插件id。
在独立模块中我们需要将插件发布一下，然后再依赖.
文件夹结构看起来像下面这样
![gradle-plugin-module](/image/gradle/gradle-plugin/gradle-plugin-with-module.png)

引入`maven`,配置一下发布信息，看起来像下面这样
``` groovy
plugins {
    id 'java-library'
    id 'kotlin'
    id 'java-gradle-plugin'
    id 'maven'
}

java {
    sourceCompatibility = JavaVersion.VERSION_1_7
    targetCompatibility = JavaVersion.VERSION_1_7
}

dependencies {
    implementation "org.jetbrains.kotlin:kotlin-stdlib:$kotlin_version"
}
uploadArchives{ //当前项目可以发布到本地文件夹中
    repositories {
        mavenDeployer {
            repository(url: uri('/Users/huangyuan/maven_repo')) //定义本地maven仓库的地址
            pom.groupId = 'com.example.mygradleplugin'   //groupId
            pom.artifactId = 'myplugin'  //artifactId
            pom.version = '1.0.2' //版本号
        }
    }
}
```
同步一下，就会看到该模块的task中多了一个upload.uploadArchives任务，执行之后会在maven仓库对应的文件夹下看到发布的jar包

2. 依赖、引用
在项目根目录中，添加一下仓库地址，然后使用classpath进行依赖
``` groovy
buildscript {
    repositories {
        //添加该maven仓库，
        maven {
            url uri('/Users/huangyuan/maven_repo')
        }
    }
    dependencies {
        //添加发布的plugin的依赖
        classpath 'com.example.mygradleplugin:myplugin:1.0.2'
    }
}

allprojects {
    repositories {
        maven {
            url uri('/Users/huangyuan/maven_repo')
        }
    }
}
```
引用和上面一样，没什么好说的


#### 进行调试
当插件工作不是预期的结果时，我们可能需要进行断点调试(当然打日志的方法也不错)，在之前的版本中还需要新增remote配置，然后以debug方式执行这个任务，现在在AndroidStudio中(4.1.1版本)中，只需要在侧边找打这个任务，右键菜单debug执行就好了

----
以上

