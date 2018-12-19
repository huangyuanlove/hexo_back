---
title: gradle-java插件
date: 2018-12-19 21:23:58
tags: [gradle,Android]
keywords: gradle基础,gradle plugin,gradle插件
---

当我们使用java插件时，只需要在gradle文件中应用`apply plugin :'java'`一下 就好了，插件中有很多默认的配置，比如源代码位置在`src/main/java`，`src/test/java`是单元测试用例的存放目录，`src/main/resources`是要打包的文件存放目录，比如配置文件和图片等。当然我们也可以改变java插件的默认配置，只需要在`build.gradle`中配置对应目录即可。

``` groovy
sourceSets{
    main{
        java{
            srcDir 'src/java'
        }
        resources{
            srcDir 'src/resources'
        }
    }
}
```

一般我们在IDEA中导入eclipse项目的时候可以暂时这样配置。

<!--more-->

#### 配置第三方依赖

要想使用三方依赖，首先要告诉Gradle从哪里找到这些依赖，一般我们从某个仓库中查找我们需要的jar包，所以我们应该配置使用什么类型的仓库：

``` groovy
repositories{
    mavenCentral()
}
```

上面配置了一个Maven中央仓库，告诉Gradle可以在Maven中央仓库中查找我们依赖的jar，此外，我们也可以从jcenter、ivy、本地Maven库、自己搭建的Maven仓库查找：

``` groovy
repositories{
    mavenCentral()
    maven{
        url "http://mymaven.com"
    }
}
```

有了仓库，我们就可以使用我们的依赖了

``` groovy
dependencies{
    compile group: 'com.squareup.okhttp3',name:'okhttp',version:'3.0.1'
}
```

或者

``` groovy
dependencies{
    compile 'com.squareup.okhttp3:okhttp:3.0.1'
}
```

此外，java插件可以为不同的源集在编译时指定不同的依赖，比如main源集指定一个编译时依赖，vip源集可以指定另外一个不同的依赖：

``` groovy
dependencies{
    mainCompile 'com.squareup.okhttp3:okhttp:3.0.1'
    vipCompile 'com.squareup.okhttp3:okhttp:2.5.0'
}
```

依赖还可以是一个子项目(module):

``` groovy
dependencies{
    compile project (':module')
}
```

依赖还可以是一个文件

``` groovy
dependencies{
    compile file ('libs/xxx.jar')
}
```

当我们依赖的jar包较多时，可以这么写

``` groovy
dependencies{
    compile fileTree(dir: 'libs', include: ['*.jar'])
}
```

#### 源码集合(SourceSet)

SourceSet--源代码集合--源集，是Java插件来描述和管理源代码及其资源的一个抽象概念，是一个Java源代码文件和资源文件的合集。

有了源集，我们就能针对不同的业务和应用对我们的源码进行分组。Java插件在Project下为我们提供了一个sourceSet属性以及一个sourceSet{}闭包来访问和配置源集

| 属性名              | 类型               | 描述                          |
| :------------------ | :----------------- | :---------------------------- |
| name                | String             | 只读，比如main                |
| output.classesDir   | File               | 该源集编译后的class文件目录   |
| output.resourcesDir | File               | 编译后生成的资源目录          |
| compileClasspath    | FileCollection     | 编译该源集时所需要的classpath |
| java                | SourceDirectorySet | 该源集的java源文件            |
| java.srcDir         | Set                | 该源集的Java源文件所在目录    |
| resources           | SourceDirectorySet | 该源集的资源文件              |
| resources.srcDir    | Set                | 该源集的资源文件所在目录      |

#### 发布构件

Gradle构建的产物，我们称之为构件，一个构件可以是一个jar包，也可以是一个zip或者war包。想要发布构件，需要先定义发布什么样的构件。下面以发布一个Jar构件为例：

``` groovy
apply plugin:'java'
task publishJar(type:Jar)
version '1.0.0'
artifacts {
    archives publishJar
}
```

发布的构件是通过`artifacts{}`闭包配置的，例子中我们通过一个Task来为我们发布提供构件，除了使用Task之外，还可以 直接发布一个文件对象：

``` groovy
def publishFile = file('build/buildile')
artifacts {
    archives publishFile
}
```

配置好需要发布的构件后就可以发布，就是把你的构件上传到一个指定的目录、仓库等

``` groovy
apply plugin:'java'

task publishJar(type:Jar)

version '1.0.0'

artifacts {
    archives publishJar
}

uploadArchives{
    repositories {
        flatDir{
            name 'libs'
            dirs "$projectDir/libs"
        }
        mavenLocal()
    }
}
```

`uploadArchives`是一个UploadTask，用于上传发布我们的构件，上面的配置是发布到我们当前项目的libs目录和本地的Maven库(mavenLocal())，当你执行`uploadArchives`任务后，可以在用户目录下的`.m2/repository`文件夹下找到它。

假如我们要上传到自己公司搭建的Nexus私服为例：

``` groovy
apply plugin:'java'
apply plugin:'maven'

task publishJar(type:Jar)

group 'com.company.projectName'
version '3.1415'

artifacts {
    archives publishJar
}

def publishFile = file('build/buildile')
artifacts {
    archives publishFile
}


uploadArchives{
    repositories {
        flatDir{
            name 'libs'
            dirs "$projectDir/libs"
        }
        mavenLocal()
        mavenDeployer{
            repository(url: "http://repo.mycompany.com/nexus/content/repositories/release"){
                authentication(userName:'userName',password:'pwd')
            }
            snapshotRepository(url: "http://repo.mycompany.com/nexus/content/repositories/snapshot"){
                authentication(userName:'userName',password:'pwd')
            }
        }
    }
}
```

这里引用了一个maven插件，它对Maven的发布构件支持的非常好，可以直接配置release和snapshot库。



----

以上