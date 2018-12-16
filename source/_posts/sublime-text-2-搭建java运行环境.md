---
title: sublime text 2 搭建java运行环境
tags: [sublime]
date: 2014-05-18 18:35:23
keywords: [sublime,sublime编译java]
---

刚开始学java，在windows下写代码的时候用sublime2写的，捣鼓了一下环境，可以进行一键编译
需要的材料：
1)  安装好的ST2
2）java运行环境
<!--more-->

#### 创建批处理或Bash Shell脚本文件
打开任意的文本编辑器，输入下面的内容，并保存为runJava.bat文件
代码如下

``` bash
@ECHO OFF  
cd %~dp1  
ECHO Compiling %~nx1.......  
IF EXIST %~n1.class (  
DEL %~n1.class  
)  
javac %~nx1  
IF EXIST %~n1.class (  
ECHO -----------OUTPUT-----------  
java %~n1  
)  
```

如果是在Windows下编译提示  不是utf-8编码什么的话，则将上诉代码修改如下

``` bash
@ECHO OFF  
cd %~dp1  
ECHO Compiling %~nx1.......  
IF EXIST %~n1.class (  
DEL %~n1.class  
)  
javac -encoding UTF-8 %~nx1  
IF EXIST %~n1.class (  
ECHO -----------OUTPUT-----------  
java %~n1  
) 
```
![geocoderSearch](/image/sublime/runJavaBat.png)

将这个文件放到JDK的bin文件夹下

####  在Sublime Text 2编辑器中配置相应的Java构建环境

打开Sublime的包目录，使用菜单Perferences->Browse Packages
![geocoderSearch](/image/sublime/browePackages.png)
选择Java目录
![geocoderSearch](/image/sublime/javacSublimeBuild.png)
打开JavaC.sublime-build，并修改下面的行  
![geocoderSearch](/image/sublime/before_modify.png)
![geocoderSearch](/image/sublime/afterModify.png.png)

画红框的是修改完之后的。
然后 Ctrl+s 保存一下就好了。
写完java代码之后，Ctrl+s 保存一下，然后Ctrl+B 就可以编译运行了。。。

----
以上
