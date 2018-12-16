---
title: gradle插件
date: 2018-12-09 23:20:59
tags: [gradle,Android]
keywords: gradle基础,gradle plugin,gradle插件
---

把插件应用到你的项目，插件会扩展项目的功能，帮助你在项目的构建过程中做很多事情。

1. 可以添加任务到你的项目中，帮你完成一些事情，比如测试、编译、打包。
2. 可以添加依赖配置到你的项目中，我们可以通过他们配置我们项目在构建过程中需要的依赖，比如变异的时候依赖第三方库等
3. 可以向项目中现有的对象类型添加新的扩展属性、方法等，让你可以使用他们帮助我们配置、优化构建，比如`android{}`这个配置块就是Android Gradle插件为Project对象添加的一个扩展。
4. 可以对项目进行一些约定，比如应用Java插件之后，约定`src/main/java`目录下使我们源代码存放位置，在编译的时候也是编译这个目录下的java源代码文件

<!-- more -->

#### 如何应用一个插件

插件的应用都是通过`Project.apply()`方法完成的，apply有好几种用法，并且插件也氛围二进制插件和脚本插件


##### 应用二进制插件

二进制插件就是实现了`org.gradle.api.Plugin`接口的插件，他们可以有`plugin id`，下面介绍如何应用一个Java插件

> apply plugin:'java'

上面的语句把Java插件应用到我们的项目中了，其中`java`是Java插件的plugin id，它是唯一的。对于Gradle自带的核心插件都有一个容易记住的短名，称其为plugin id,比如这里的java，其实它对应的类型是`org.gradle.api.plugins.JavaPlugin`，所以通过该类型我们也可以应用这个插件

> apply plugin: org.gradle.api.plugins.JavaPlugin

又因为包`org.gradle.api.plugins`是默认导入的，所以可以去掉包名直接写为：

> apply plugin:JavaPlugin

以上三种写法是等价的，第二种写法一般适用于我们在build文件中自定义的插件，也就是脚本插件。

##### 应用脚本插件

build.gradle

``` groovy
apply from :'version.gradle'
task test<<{
    println "app版本是${versionName},版本号是${versionCode}"
}
```

version.gradle

``` :jack_o_lantern:
ext{
    versionName="1.0.0"
    versionCode=1
}
```

其实这不能算是一个插件它只是一个脚本，应用脚本插件，其实就是把这个脚本加载进来，和二进制插件不同的是它使用`from`关键字,后面紧跟的是一个脚本文件，可以使本地的，也可以使网络存在的，如果是网络上的话要使用http url

##### apply方法的其他用法

Project.apply()方法有3中使用方式，它们只是接受的参数不同，上面使用的是接受一个Map类型参数的形式，下面是其他两种方式

``` groovy
void apply(Map<String,?> options);
void apply(Closure closure);
void apply(Action<? super ObjectConfigurationAction> action)
```

闭包的方式如下：

``` groovy
apply{
    plugin 'java'
}
```

该闭包被用来配置一个`ObjectConfigurationAction`对象，所以可以在比暴力使用ObjectConfigurationAction对象的方法、属性等进行配置。

Action的方式

``` groovy
apply(new Action<ObjectConfigurationAction>(){
    @Override
    void execute(ObjectConfigurationAction objectConfigurationAction){
        objectConfigurationAction.plugin('java')
    }
})
```

Action的方式需要new一个Action，然后在execute方法里进行配置

##### 应用第三方发布的插件

在使用第三方发布的插件的时候，必须要在buildscript{}里配置其classpath才能使用，比如在使用Android Gradle插件，就属于Android发布的第三方插件：

``` groovy
buildscript{
    repositories{
        jcenter()
    }
    dependencies{
        classpath 'com.android.application'
    }
}

```

buildscript{}块是一个在构建项目之前，为项目进行前期准备和初始化相关配置依赖的地方，配置好所需的依赖，就可以应用插件了：

``` groovy
apply plugin :'com.android.application'
```

如果没有提前在buildscript里配置依赖的classpath，会提示找不到这个插件

