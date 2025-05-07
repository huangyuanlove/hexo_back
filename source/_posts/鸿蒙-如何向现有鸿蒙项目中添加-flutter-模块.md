---
title: 鸿蒙-如何向现有鸿蒙项目中添加 flutter 模块
tags: [HarmonyOS]
date: 2025-04-29 10:37:47
keywords: HarmonyOS,鸿蒙应用开发,跨平台,flutter,鸿蒙支持flutter,鸿蒙添加flutter模块
---

## 前言
在版本迭代时，有部分新增的功能，需要开发新的业务模块，这里希望使用跨平台框架，实现代码Android、iOS、HarmonyOS 复用。之前的部分业务使用flutter 开发，HarmonyOS 也支持 flutter 的接入，因此，这次依然使用选择使用 flutter。

## 环境
可以支持运行在HarmonyOS 系统上的 flutter 仓库已经迁移至GitCode平台，目前有两个版本：基于 flutter 3.22.0 版本和基于 flutter3.7.12 版本。下面的仓库链接是 3.22.0 版本
> 
gitcode分支（官方分支，持续更新）：
https://gitcode.com/openharmony-sig/flutter_flutter/tree/3.22.0-ohos
原始仓库
https://gitee.com/harmonycommando_flutter/flutter
Engine:
gitcode分支（官方分支，持续更新）：
https://gitcode.com/openharmony-sig/flutter_engine/tree/oh-3.22.0
原始仓库
https://gitee.com/harmonycommando_flutter/flutter_engine


我们当前使用的是 基于flutter 3.7.12的版本，并且使用fvm 来管理多个flutter 版本。

``` shell
# 本地使用fvm进行flutter版本管理，所以将harmony-flutter也放到fvm内管理
cd /Users/xxx/fvm/versions
# clone鸿蒙flutter代码
git clone https://gitee.com/openharmony-sig/flutter_flutter.git 3.7.12-ohos
git checkout -b dev origin/dev
# 将本地flutter版本改为鸿蒙flutter
fvm global 3.7.12-ohos
# 检查配置是否正确，Flutter版本 & HarmonyOS
flutter doctor -v
 
```
我们看到 HarmonyOS 的支持，配置环境是成功的。

## 集成

### 创建 flutter 模块
在鸿蒙项目文件夹中创建flutter module

``` shell
# 为方便代码管理，将flutter代码放到工程目录内统一管理
cd HarmonyProject
# 1.以module形式集成到项目，创建flutter_module
flutter create -t module flutter_module
# 2.构建har文件
cd flutter_module
flutter build har --release
``` 
### 引用 flutter

我们可以在开发阶段引用源码，在测试发版时引用 har 文件
修改鸿蒙工程根目录下的`oh-package.json5`文件并添加对应的依赖
``` json5
"dependencies":{
    "@ohos/flutter_ohos": "file:./flutter_module/.ohos/har/flutter.har",
    "@ohos/flutter_module": "flutter_module/.ohos/flutter_module"
}

```
在测试发版的时候先将 flutter 模块打成 har 包，复制到鸿蒙项目中，直接引用 har 文件就好。

``` shell
# 2.构建har文件
cd flutter_module
flutter build har --release
```
这里有个小问题，flutter 项目引入了三方库时，会生成多个 har 文件，在鸿蒙工程内运行时会报错，
我们可以添加`overrides`声明
``` json
"dependencies":{

},
"overrides":{
    "@ohos/flutter_ohos": "file:./har/flutter.har",
    "fluttertoast": "file:har/fluttertoast.har",
    "path_provider_ohos": "file:har/path_provider_ohos.har",
    "permission_handler_ohos": "file:har/permission_handler_ohos.har"
}
```

## 交互
