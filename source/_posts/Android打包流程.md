---
title: Android打包流程
tags: [Android,gradle]
date: 2020-11-11 22:12:20
keywords: [Android打包流程,Android构建]
---
现在的Android开发大部分是在AndroidStudio中进行的，当我们想要得到APK文件的时候，点一下RUN，或者执行一下`gradlew assembleDebug` 就可以了，那么在这个过程中到底发生了什么，我们来详细看一下。
构建的过程大致可以划分为两个过程：编译和打包
编译：编译器(compileer)通过编译source code、AIDL files、source filse、dependednce files，最终生成Dex(s)文件和编译后的资源文件
打包：打包器(APK packager)利用签名文件(KeyStore)和上一步编译过程中生成的Dex(s)文件、编译后的资源文件打包成最终的APK文件
<!--more-->
#### 构建过程
一个几乎是最简化的构建流程
![最简化的构建流程](/image/gradle/apk打包流程--最简.png  "最简化的构建流程")
上图中的菱形表示一些构建操作，矩形表示输入或者输出文件


#### 初见细节

![稍微有点细节的构建流程](/image/gradle/apk打包流程.png  "稍微有点细节的构建流程")

编译打包流程
1. 使用aapt/aapt2编译资源文件生成resource.arsc和R.java

2. 使用AIDL处理aidl文件，生成java文件

3. 使用javac编译java文件，生成class文件

4. (proguard混淆如果有)使用DX/D8/R8处理class文件，生成最终需要的dex文件

5. 使用Android NDK处理native代码生成.so文件

6. 使用apkbuilder生成未签名的apk文件

7. 使用apksigner对apk进行签名，生成签名后的apk文件

8. 使用zipalign工具，对已签名的apk文件进行优化(只有v1签名才有这一步，v2签名的apk会在zipalign后签名被破坏）。



#### 终章

这是一张流传已久的网图，找到的图已经有包浆了，又重新画了一遍
在图的最下方有示例说明：
1. 矩形表示文件
2. 椭圆表示工具
3. 箭头表示输出
4. 空心圆表示输入

![最终构建流程](/image/gradle/apk打包流程final.png  "最终构建流程")

既然都是调用的工具，我们同样可以自己写脚本运行这些工具进行打包，调用的工具详细信息可以在这里找到 [https://developer.android.com/studio/command-line?hl=zh_cn](https://developer.android.com/studio/command-line?hl=zh_cn)


文中图片的源文件在 /image/apk打包流程.drawio ,使用drawio绘制;

----

以上

参考：
1. https://www.zhihu.com/search?type=content&q=Android%E6%89%93%E5%8C%85%E6%B5%81%E7%A8%8B
2. https://juejin.im/post/6882328361294069773

