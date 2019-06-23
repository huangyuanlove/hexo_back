---
title: Android接入flutter多渠道引起的问题
tags: [Android,Flutter,Android爬坑之旅]
date: 2019-06-19 11:02:19
keywords: [Android,Flutter,多渠道]
---

公司已经开始在项目中使用Flutter进行跨平台开发了，并且已经在其中一款应用上线了，过程并没有多曲折，按照官网教程一步步进行，继承、打包、测试、发版。

最近另外一个项目有新的需求，也开始使用Flutter，在集成打包的时候出现了问题：

`A/flutter: [FATAL:flutter/shell/common/shell.cc(218)] Check failed: vm. Must be able to initialize the VM.`

收集到的错误日志如下：

<!--more-->


``` crash
E/CrashReport: #++++++++++Record By Bugly++++++++++#
E/CrashReport: # You can use Bugly(http:\\bugly.qq.com) to get more Crash Detail!
E/CrashReport: # PKG NAME: me.chunyu.ChunyuDoctor
E/CrashReport: # APP VER: 8.6.0
E/CrashReport: # LAUNCH TIME: 2019-06-19 18:00:12
E/CrashReport: # CRASH TYPE: NATIVE_CRASH
E/CrashReport: # CRASH TIME: 2019-06-19 18:00:22
E/CrashReport: # CRASH PROCESS: me.chunyu.ChunyuDoctor
E/CrashReport: # CRASH THREAD: main(2)
E/CrashReport: # REPORT ID: d88b5d1a-cbb0-4920-b41c-d8ea8eb804bd
E/CrashReport: # CRASH DEVICE: MI 5 UNROOT
E/CrashReport: # RUNTIME AVAIL RAM:1411100672 ROM:11107233792 SD:11107233792
E/CrashReport: # RUNTIME TOTAL RAM:2824822784 ROM:26300211200 SD:26300211200
E/CrashReport: # EXCEPTION FIRED BY UNKNOWN_USER me.chunyu.ChunyuDoctor(27335)
E/CrashReport: # CRASH STACK: 
E/CrashReport: SIGABRT
    0x6ac7
    #00    pc 0004b10c    /system/lib/libc.so (tgkill+12) [armeabi-v7a::d7f479b7abcbff3bc09ca6d100c44333]
    #01    pc 0001a9a3    /system/lib/libc.so (abort+54) [armeabi-v7a::d7f479b7abcbff3bc09ca6d100c44333]
    #02    pc 00af85ab    /data/app/me.chunyu.ChunyuDoctor-n6cyA4a8iSyIl6wZ_2ghlw==/lib/arm/libflutter.so [armeabi-v7a::a12434e0b53806a35730000001000000]
    #03    pc 00b197cf    /data/app/me.chunyu.ChunyuDoctor-n6cyA4a8iSyIl6wZ_2ghlw==/lib/arm/libflutter.so [armeabi-v7a::a12434e0b53806a35730000001000000]
    #04    pc 00aeaf45    /data/app/me.chunyu.ChunyuDoctor-n6cyA4a8iSyIl6wZ_2ghlw==/lib/arm/libflutter.so [armeabi-v7a::a12434e0b53806a35730000001000000]
    #05    pc 00af0507    /data/app/me.chunyu.ChunyuDoctor-n6cyA4a8iSyIl6wZ_2ghlw==/lib/arm/libflutter.so [armeabi-v7a::a12434e0b53806a35730000001000000]
    #06    pc 00aef79b    /data/app/me.chunyu.ChunyuDoctor-n6cyA4a8iSyIl6wZ_2ghlw==/lib/arm/libflutter.so [armeabi-v7a::a12434e0b53806a35730000001000000]
    #07    pc 003d9d29    /system/lib/libart.so (ExecuteMterpImpl+37417) [armeabi-v7a::d2078ba50169919802b98e67fdc80c58]
    #08    pc 003d5fe1    /system/lib/libart.so (ExecuteMterpImpl+21729) [armeabi-v7a::d2078ba50169919802b98e67fdc80c58]
    #09    pc 003da555    /system/lib/libart.so (art_quick_invoke_stub+228) [armeabi-v7a::d2078ba50169919802b98e67fdc80c58]
    #10    pc 000a0e1d    /system/lib/libart.so (_ZN3art9ArtMethod6InvokeEPNS_6ThreadEPjjPNS_6JValueEPKc+140) [armeabi-v7a::d2078ba50169919802b98e67fdc80c58]
    #11    pc 001e6e53    /system/lib/libart.so (_ZN3art11interpreter34ArtInterpreterToCompiledCodeBridgeEPNS_6ThreadEPNS_9ArtMethodEPKNS_7DexFile8CodeItemEPNS_11ShadowFrameEPNS_6JValueE+238) [armeabi-v7a::d2078ba50169919802b98e67fdc80c58]
    #12    pc 001e2403    /system/lib/libart.so (_ZN3art11interpreter6DoCallILb0ELb0EEEbPNS_9ArtMethodEPNS_6ThreadERNS_11ShadowFrameEPKNS_11InstructionEtPNS_6JValueE+574) [armeabi-v7a::d2078ba50169919802b98e67fdc80c58]
    #13    pc 003c0d75    /system/lib/libart.so (MterpInvokeDirect+360) [armeabi-v7a::d2078ba50169919802b98e67fdc80c58]
    #14    pc 003c8394    /system/lib/libart.so (artInvokeSuperTrampolineWithAccessCheck+3599) [armeabi-v7a::d2078ba50169919802b98e67fdc80c58]
    #15    pc 001c9985    /system/lib/libart.so (_ZN3art22IndirectReferenceTable10VisitRootsEPNS_11RootVisitorERKNS_8RootInfoE+16) [armeabi-v7a::d2078ba50169919802b98e67fdc80c58]
    #16    pc 001ced07    /system/lib/libart.so (_ZN3art11interpreter33ArtInterpreterToInterpreterBridgeEPNS_6ThreadEPKNS_7DexFile8CodeItemEPNS_11ShadowFrameEPNS_6JValueE+142) [armeabi-v7a::d2078ba50169919802b98e67fdc80c58]
    #17    pc 001e23ed    /system/lib/libart.so (_ZN3art11interpreter6DoCallILb0ELb0EEEbPNS_9ArtMethodEPNS_6ThreadERNS_11ShadowFrameEPKNS_11InstructionEtPNS_6JValueE+552) [armeabi-v7a::d2078ba50169919802b98e67fdc80c58]
    #18    pc 003bfebf    /system/lib/libart.so (MterpInvokeVirtual+446) [armeabi-v7a::d2078ba50169919802b98e67fdc80c58]
    #19    pc 003c8294    /system/lib/libart.so (artInvokeSuperTrampolineWithAccessCheck+3343) [armeabi-v7a::d2078ba50169919802b98e67fdc80c58]
    #20    pc 001c9985    /system/lib/libart.so (_ZN3art22IndirectReferenceTable10VisitRootsEPNS_11RootVisitorERKNS_8RootInfoE+16) [armeabi-v7a::d2078ba50169919802b98e67fdc80c58]
    #21    pc 001ced07    /system/lib/libart.so (_ZN3art11interpreter33ArtInterpreterToInterpreterBridgeEPNS_6ThreadEPKNS_7DexFile8CodeItemEPNS_11ShadowFrameEPNS_6JValueE+142) [armeabi-v7a::d2078ba50169919802b98e67fdc80c58]
    #22    pc 001e23ed    /system/lib/libart.so (_ZN3art11interpreter6DoCallILb0ELb0EEEbPNS_9ArtMethodEPNS_6ThreadERNS_11ShadowFrameEPKNS_11InstructionEtPNS_6JValueE+552) [armeabi-v7a::d2078ba50169919802b98e67fdc80c58]
    #23    pc 003c0d75    /system/lib/libart.so (MterpInvokeDirect+360) [armeabi-v7a::d2078ba50169919802b98e67fdc80c58]
    #24    pc 003c8394    /system/lib/libart.so (artInvokeSuperTrampolineWithAccessCheck+3599) [armeabi-v7a::d2078ba50169919802b98e67fdc80c58]
    #25    pc 001c9985    /system/lib/libart.so (_ZN3art22IndirectReferenceTable10VisitRootsEPNS_11RootVisitorERKNS_8RootInfoE+16) [armeabi-v7a::
E/CrashReport: #++++++++++++++++++++++++++++++++++++++++++#
```

刚开始以为是so包的问题，但是用到so的地方和已经发版的应用没啥区别。排除这个问题的影响。
网上疯狂搜索，看到了这个
https://www.jianshu.com/p/273c462896c3
说是因为安装包缺少flutter_assets导致的。解开安装包，发现现象一致。因为多渠道打包的问题，导致`flutter_assets`没有被打包进去，flutter的Issues#27027 https://github.com/flutter/flutter/issues/27028  有提到这个，需要在fluuter module的build.gradle(flutter module/.android/Flutter/build.gradle)中配置同样的渠道信息。

![问题原因](/image/flutter/flutter_muilty_flavor.png)

配置完了打包安装运行还是包上面的错误。根据上面文章，找到flutter安装目录下的打包使用的gradle,`flutter/packages/flutter_tools/gradle/flutter.gradle`文件，其中有这么个方法

`private void addFlutterTask(Project project)`

![copyFlutterAssetsTask](/image/flutter/copy_flutter_assets_task.png)

其中定义了合并资源的task：` Task mergeAssets = project.tasks.findByPath(":${mainModuleName}:merge${variant.name.capitalize()}Assets")`

这里面的变量`mainModuleName`默认是`app`,但是我们的工程的主module不是这个名字，这个task就不会被执行。

加上打印日志，果然`mergeAssets `是null，导致资源文件不会被打包进去。

这个问题有人发现了，并且提交了 Pull request,地址 <https://github.com/flutter/flutter/pull/27154>

解决方案是在我们工程的`setting.gradle`文件中指定主module的名字

`setBinding(new Binding([gradle: this, mainModuleName: 'example']))`



关于上面的多渠道打包的问题，有人说flutter module中的.android文件夹是flutter自己生成的，提交代码的时候并不会提交这个文件夹下的任何内容。我们可以魔改一下上面提到的`flutter.gradle`文件，在打多渠道包的时候将当前的渠道信息传进去，在gradle文件中读取一下渠道信息就好。

这里需要提一下AS生成的合并资源任务名字：比如当前渠道是`normal`,主工程的名字是`example`,生成的任务则是`exampleMergeNormalDebugAssets`和`exampleMergeNormalReleaseAssets`。可以在AS中打开gradle任务预览，在flutter这个module中`Task-->other`分组中找到该任务。


``` groovy
String flavor = ""
try{
    String tempFlavor = project.rootProject.ext.app.flavor
    if(tempFlavor!=null && !tempFlavor.empty){
        flavor = tempFlavor
    }
    
} catch (Exception e){
    println e
}

Task mergeAssets

if(flavor!=null && !flavor.empty){
    mergeAssets = project.tasks.findByPath(":${mainModuleName}:merge${flavor.capitalize()}${variant.name.capitalize()}Assets")
}else{
    mergeAssets = project.tasks.findByPath(":${mainModuleName}:merge${variant.name.capitalize()}Assets")
}
```



----

以上