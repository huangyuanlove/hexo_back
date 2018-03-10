---
title: Tinker初探
date: 2018-03-10 12:35:18
tags: [Android,热修复，tinker]
---
前两天想试一下热修复的功能，对比各大平台的热修复功能，看到tinker的[文档介绍](http://www.tinkerpatch.com/Docs/intro)，最终决定先拿Tinker试一下。
> 
||||||
|:---:|:---:|:---:|:---:|:---:
||Tinker|QZone|AndFix|Robust|
|类替换|yes|yes|no|no|
|So替换|yes|no|no|no|
|资源替换|yes|yes|no|no|
|全平台支持|yes|yes|no|yes|
|即时生效|no|no|yes|yes|
|性能损耗|较小|较大|较小|较小|
|补丁包大小|较小|较大|一般|一般|
|开发透明|yes|yes|no|no|
|复杂度|较低|较低|复杂|复杂|
|Rom体积|Dalvik较大|较小|较小|较小|
|成功率|较高|较高|一般|最高|
>Tinker热补丁方案不仅支持类、So 以及资源的替换，它还是2.X－7.X的全平台支持。利用Tinker我们不仅可以用做 bugfix,甚至可以替代功能的发布。Tinker 已运行在微信的数亿 Android 设备上，那么为什么你不使用 Tinker 呢？

不得不说，我真的低估了跟着腾讯文档走的难度。
<!--more-->

### 注册 TinkerPatch 平台

因为需要下发补丁，直接使用TinkerPatch平台就好，在这里注册[http://www.tinkerpatch.com/Index/reg](http://www.tinkerpatch.com/Index/reg),注册完成后创建一个应用，拿到`appKey`
然后添加一个APP版本

### SDK接入
测试成功的工程全部文件在这里[https://github.com/huangyuanlove/TestTinker](https://github.com/huangyuanlove/TestTinker),包含构建成功之后的apk文件以及一些辅助文件。
##### 添加Gradle插件依赖
AndroidStudio创建一个工程，定义使用的SDK版本，我是放在了`gradle.properties` 这个文件中，
> TINKER_VERSION=1.9.2
TINKERPATCH_VERSION=1.2.2

然后在工程的`build.gradle`文件中添加插件依赖
> classpath "com.tinkerpatch.sdk:tinkerpatch-gradle-plugin:${TINKERPATCH_VERSION}"

然后添加一些其他配置，整个文件内容如下
``` groovy
buildscript {
    repositories {
        google()
        jcenter()
    }
    dependencies {
        classpath 'com.android.tools.build:gradle:3.0.1'
        classpath "com.tinkerpatch.sdk:tinkerpatch-gradle-plugin:${TINKERPATCH_VERSION}"
    }
}
if (JavaVersion.current().isJava8Compatible()) {
    allprojects {
        tasks.withType(Javadoc) {
            options.addStringOption('Xdoclint:none', '-quiet')
        }
    }
}
subprojects {
    tasks.withType(JavaCompile) {
        sourceCompatibility = JavaVersion.VERSION_1_7
        targetCompatibility = JavaVersion.VERSION_1_7
    }
}
allprojects {
    repositories {
        google()
        jcenter()
    }
}
task clean(type: Delete) {
    delete rootProject.buildDir
}

```
##### 集成 TinkerPatch SDK
在`app/build.gradle`里面添加依赖
> 
    annotationProcessor("com.tinkerpatch.tinker:tinker-android-anno:${TINKER_VERSION}") { changing = true }
    compileOnly("com.tinkerpatch.tinker:tinker-android-anno:${TINKER_VERSION}") { changing = true }
    implementation("com.tinkerpatch.sdk:tinkerpatch-android-sdk:${TINKERPATCH_VERSION}") { changing = true }

为了配置方便，我们把TinkerPatchSupport相关的配置放在一个单独的gradle文件中，在app下创建一个`tinkerpatch.gradle`，我们需要在`app/build.grale`文件中引用
> apply from: 'tinkerpatch.gradle'

##### 配置 tinkerpatchSupport 参数
编辑 `app/tinkerpatch.gralde`文件
``` groovy

apply plugin: 'tinkerpatch-support'

/**
 * TODO: 请按自己的需求修改为适应自己工程的参数
 */
def bakPath = file("${buildDir}/bakApk/")
def baseInfo = "app-1.0.0-0309-21-30-56" //构建差异文件时使用
def variantName = "debug"

/**
 * 对于插件各参数的详细解析请参考
 * http://tinkerpatch.com/Docs/SDK
 */
tinkerpatchSupport {
    /** 可以在debug的时候关闭 tinkerPatch **/
    /** 当disable tinker的时候需要添加multiDexKeepProguard和proguardFiles,
        这些配置文件本身由tinkerPatch的插件自动添加，当你disable后需要手动添加
        你可以copy本示例中的proguardRules.pro和tinkerMultidexKeep.pro,
        需要你手动修改'tinker.sample.android.app'本示例的包名为你自己的包名, com.xxx前缀的包名不用修改
     **/
    tinkerEnable = true
    reflectApplication = false
    /**
     * 是否开启加固模式，只能在APK将要进行加固时使用，否则会patch失败。
     * 如果只在某个渠道使用了加固，可使用多flavors配置
     **/
    protectedApp = false
    /**
     * 实验功能
     * 补丁是否支持新增 Activity (新增Activity的exported属性必须为false)
     **/
    supportComponent = true

    autoBackupApkPath = "${bakPath}"

    appKey = "2b662623551153ee"

    /** 注意: 若发布新的全量包, appVersion一定要更新 **/
    appVersion = "1.0.0"

    def pathPrefix = "${bakPath}/${baseInfo}/${variantName}/"
    def name = "${project.name}-${variantName}"

    baseApkFile = "${pathPrefix}/${name}.apk"
    baseProguardMappingFile = "${pathPrefix}/${name}-mapping.txt"
    baseResourceRFile = "${pathPrefix}/${name}-R.txt"

    /**
     *  若有编译多flavors需求, 可以参照： https://github.com/TinkerPatch/tinkerpatch-flavors-sample
     *  注意: 除非你不同的flavor代码是不一样的,不然建议采用zip comment或者文件方式生成渠道信息（相关工具：walle 或者 packer-ng）
     **/
}

/**
 * 用于用户在代码中判断tinkerPatch是否被使能
 */
android {
    defaultConfig {
        buildConfigField "boolean", "TINKER_ENABLE", "${tinkerpatchSupport.tinkerEnable}"
    }
}

/**
 * 一般来说,我们无需对下面的参数做任何的修改
 * 对于各参数的详细介绍请参考:
 * https://github.com/Tencent/tinker/wiki/Tinker-%E6%8E%A5%E5%85%A5%E6%8C%87%E5%8D%97
 */
tinkerPatch {
    ignoreWarning = false
    useSign = true
    dex {
        dexMode = "jar"
        pattern = ["classes*.dex"]
        loader = []
    }
    lib {
        pattern = ["lib/*/*.so"]
    }

    res {
        pattern = ["res/*", "r/*", "assets/*", "resources.arsc", "AndroidManifest.xml"]
        ignoreChange = []
        largeModSize = 100
    }

    packageConfig {
    }
    sevenZip {
        zipArtifact = "com.tencent.mm:SevenZip:1.1.10"
    }
    buildConfig {
        keepDexApply = false
    }
}
```
每个参数的含义如下

|参数|默认值|描述|
|:----:|:----:|:----:|
|tinkerEnable|true|是否开启 tinkerpatchSupport 插件功能|
|appKey	|""	|在 TinkerPatch 平台 申请的 appkey|
|appVersion	|""	|在 TinkerPatch 平台 输入的版本号,注意，我们使用 appVersion 作为 TinkerId, 我们需要保证每个发布出去的基础安装包的 appVersion 都不一样。|
|reflectApplication|false|是否反射 Application 实现一键接入；一般来说，接入 Tinker 我们需要改造我们|的 Application, 若这里为 true， 即我们无需对应用做任何改造即可接入。|
|autoBackupApkPath|""|将每次编译产生的 apk/mapping.txt/R.txt 归档存储的位置|
|baseApkFile|""|基准包的文件路径, 对应 tinker 插件中的 oldApk 参数;编译补丁包时，必需指定基准版本的 apk，默认值为空，则表示不是进行补丁包的编译。|
|baseProguardMappingFile|""|基准包的 Proguard mapping.txt 文件路径, 对应 tinker 插件 applyMapping 参数；在编译新的 apk 时候，我们希望通过保持基准 apk 的 proguard 混淆方式，从而减少补丁包的大小。这是强烈推荐的，编译补丁包时，我们推荐输入基准 apk 生成的 mapping.txt 文件。|
|baseResourceRFile|""|基准包的资源 R.txt 文件路径, 对应 tinker 插件 applyResourceMapping 参数；在编译新的apk时候，我们希望通基准 apk 的 R.txt 文件来保持 Resource Id 的分配，这样不仅可以减少补丁包的大小，同时也避免由于 Resource Id 改变导致 remote view 异常。|
|protectedApp	|false	|是否开启支持加固，注意：只有在使用加固时才能开启此开关|
|supportComponent	|false	|是否开启支持在补丁包中动态增加Activity 注意：新增Activity的Exported属性必须为false|
|backupFileNameFormat|	'\${appName}-\${variantName}'|	格式化命名备份文件 这里请使用单引号|

##### 初始化 TinkerPatch SDK
这里推荐使用改造之后的ApplicationLike，对应`tinkerpatch.gradle`文件中的`reflectApplication = false`,这里给出了完整的ApplicationLike类，可以在这里查看[https://github.com/huangyuanlove/TestTinker/blob/master/app/src/main/java/com/huangyuan/testtinker/SampleApplicationLike.java](https://github.com/huangyuanlove/TestTinker/blob/master/app/src/main/java/com/huangyuan/testtinker/SampleApplicationLike.java)
其中对于类的注解中的 `application` 的值，就是我们应用的Application类，需要在`AndroidManifest.xml`中的`application`标签中配置
``` java
@DefaultLifeCycle(application = "com.huangyuanlove.testtinker.SampleApplication",
                  flags = ShareConstants.TINKER_ENABLE_ALL,
                  loadVerifyFlag = false)`
```

**注意：初始化的代码建议紧跟 super.onCreate(),并且所有进程都需要初始化，已达到所有进程都可以被 patch 的目的
如果你确定只想在主进程中初始化 tinkerPatch，那也请至少在 :patch 进程中初始化，否则会有造成 :patch 进程crash，无法使补丁生效**
我们在实际应用过程中，可以在登陆等关键地方去调用`TinkerPatch.with().fetchPatchUpdate(true)`来检测有没有新的补丁包，若有，则去下载。下载完成补丁包后，sdk会自动去合成新的安装包，并且在息屏的时候自动重启主线程去加载新的文件，或者调用` ShareTinkerInternals.killAllOtherProcess(getApplicationContext());
                android.os.Process.killProcess(android.os.Process.myPid());`来完成杀死主线程的目的。
##### 使用步骤
首先构建基础包，模拟用户当前使用的版本。
在gradle中找到下图所示的 `assembleRelease`或者`assembleDebug`task，需要注意的是，如果构建基础包使用的是`debug`,那么在构建patch包的时候也要选择`debug`，还有就是尽量把`app/tinkerpatch.gradle`中定义的`variantName`改成一致的。
基础包构建成功后，会在`app/build/bakApk`文件夹下生成对应的文件，找到和你构建时间一致的包。
现在修改代码或者布局文件(模拟修复bug),修改清单文件`AndroidManifest.xml`中的versionName和versionCode。
修改`app/tinkerpatch.gradle`文件，将其中定义的`baseInfo`修改为上面提到的路径。这时候**不需要修改**该文件中的`appVersion`。
在gradle中找到tinker任务包，找到`tinkerPatchDebug`或者`tinkerPatchRelease`，构建差异包(补丁文件)。构建成功后会在`app/build/outputs/apk/tinkerPatch`文件夹中
![tinkerTaskResult](/image/hotfix/tinker_task.png) ![tinkerTaskResult](/image/hotfix/tinker_task_result.png)
现在我们已经成功构建的差异包`patch-signed-7zip.apk`,现在只需要将差异包上传到`tinker-patch`平台就可以了。

##### 在tinker-patch平台发布差异包
我们登陆tinker-patch平台，找到在刚开始创建的项目，在该项目里面添加一个App版本，注意这里的App版本号要和`tinkerpatch.gradle`里面定义的`appVersion`一致，在官方文档中也提到过这一点：
>每一个 APP 版本对应一个已经发布的 base apk, 这里我们可以使用 APP 版本作为 TinkerID。我们需要保证每个发布的 APK 都采用不用的 APP 版本。

创建好app版本之后，点击`发布新补丁`，选择补丁文件`patch-signed-7zip.apk`,填写一下备注就好了，这里有四种补丁的下发方式[开发预览](http://www.tinkerpatch.com/Docs/dev) 、`全量下发` 、[条件下发](http://www.tinkerpatch.com/Docs/rule) 、[灰度下发](http://www.tinkerpatch.com/Docs/rule)、具体差异可以点击去查看。
同时我们也可以在平台对应的软件版本中的实时监控里面看到补丁的下载以及合成应用次数。

----
以上