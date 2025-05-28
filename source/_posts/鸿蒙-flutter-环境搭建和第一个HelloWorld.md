---
title: 鸿蒙-flutter-环境搭建和第一个HelloWorld
tags: [HarmonyOS，Flutter]
date: 2025-05-27 15:59:28
keywords: HarmonyOS,鸿蒙应用开发,鸿蒙手机应用开发,flutter
---

## 前言
正在慢慢的补齐鸿蒙版本应用的功能，之前 Android 和 iOS 上有一部分功能是 flutter 实现的，现在需要把相关代码移植到鸿蒙应用中。慢慢来，不着急。
由于目前只有64位引擎，暂不支持模拟器，需要使用真机调试。

## 环境
现存的flutter 相关代码使用的是 flutter3.0.2 版本，正好趁这个机会升级一下版本。
由于鸿蒙版的flutter 3.22.0 已经 release，直接升级到这个版本。
这里插播一条消息
> 所有鸿蒙相关开源仓后续均迁移至GitCode平台，SIG、TPC组织已完成迁移，OpenHarmony主组织仓也即将完成迁移
SDK:
gitcode分支（官方分支，持续更新）：
https://gitcode.com/openharmony-sig/flutter_flutter/tree/3.22.0-ohos
原始仓库
https://gitee.com/harmonycommando_flutter/flutter
Engine:
gitcode分支（官方分支，持续更新）：
https://gitcode.com/openharmony-sig/flutter_engine/tree/oh-3.22.0
原始仓库
https://gitee.com/harmonycommando_flutter/flutter_engine

### 鸿蒙环境
鸿蒙的环境很简单，下载最新的 DevEco 安装好就行了。这样就包含了开发鸿蒙所需要的所有工具。
但使用 flutter 混编时还需要将几个工具路径添加的环境变量里面。
当然了，不下载 DevEco，下载对应的`Command Line Tools for HarmonyOS `也可以。但是免不了会编写一些 ArkTS 相关的鸿蒙代码，比如录音、相机的调用等，目前来看还没有其他的IDE 支持 ArkTS 的语法，所以，还是建议安装 DevEco。
需要添加到环境变量的工具。
mac上需要配置这些
``` shell
 export TOOL_HOME=/Applications/DevEco-Studio.app/Contents # mac环境
 export DEVECO_SDK_HOME=$TOOL_HOME/sdk # command-line-tools/sdk
 export PATH=$TOOL_HOME/tools/ohpm/bin:$PATH # command-line-tools/ohpm/bin
 export PATH=$TOOL_HOME/tools/hvigor/bin:$PATH # command-line-tools/hvigor/bin
 export PATH=$TOOL_HOME/tools/node/bin:$PATH # command-line-tools/tool/node/bin
```
在 windows 上有些修改，
需要配置一个变量名为`HOS_SDK_HOME`,值为sdk路径的变量，比如我是安装的DevEco，sdk路径就是`D:\DevEco\DevEco Studio\sdk`
![](image/harmony_flutter/HOS_SDK_HOME.png)。
然后我们再将`D:\DevEco\DevEco Studio\tools\hvigor\bin`、`D:\DevEco\DevEco Studio\tools\node`、`D:\DevEco\DevEco Studio\tools\ohpm\bin`添加到环境变量。

### flutter 环境
克隆(flutter_flutter)[https://gitcode.com/openharmony-tpc/flutter_flutter/tree/3.22.0-ohos]仓库的`3.22.0-ohos`分支就可以。
``` shell
git clone -b 3.22.1-ohos-1.0.1 https://gitcode.com/openharmony-sig/flutter_flutter.git
```
下载完成后，将`flutter`添加到环境变量  
mac环境
``` shell
 export PATH=<flutter_flutter path>/bin:$PATH
 export PUB_HOSTED_URL=https://pub.flutter-io.cn  #国内的镜像，也可以使用其他镜像，比如清华镜像源
 export FLUTTER_STORAGE_BASE_URL=https://storage.flutter-io.cn #国内的镜像，也可以使用其他镜像，比如清华镜像源
```

windows环境变量，也是将`<flutter_flutter path>\bin`添加到环境变量。然后再分别新建`PUB_HOSTED_URL`、`FLUTTER_STORAGE_BASE_URL`添加到环境变量。

### 检查
命令行执行一下`flutter doctor -v`,如果能找到能执行成功，并且Futter与OpenHarmony应都为ok标识，若两处提示缺少环境，按提示补上相应环境即可。

![harmony_flutter_doctor_mac](image/harmony_flutter/harmony_flutter_doctor_mac.png)


## 第一个 HelloWorld
创建项目的命令和官方的 flutter 是一样的，只不过是多了一个鸿蒙平台的支持。
我们创建一个支持 Android、iOS 和鸿蒙的 flutter 项目。
``` shell
flutter create  --platforms android,ios,ohos  -i objc -a java ohflutter_3221
```
可以指定 iOS 平台使用 oc,Android平台使用 Java 语言，鸿蒙平台不需要指定，只有一个 ArkTS 可用
执行的结果也没什么两样,只不过在对应的项目文件夹下多了一个`ohos`文件夹，和 `android`、`ios`文件夹一样，用来存放原生相关的代码
然后我们连接好设备，在工程文件夹下执行`flutter run`，会提示我们需要配置调试签名。
![oh_flutter_config_sign](image/harmony_flutter/oh_flutter_config_sign.png)。
这里我们需要注册一个华为开发者账号，然后按照提示进行签名。
配置完签名之后，我们再次执行`flutter run`

看下效果
![运行结果](image/harmony_flutter/oh_flutter_hello_world.gif)


