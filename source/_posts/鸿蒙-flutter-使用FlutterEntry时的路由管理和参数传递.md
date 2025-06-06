---
title: 鸿蒙-flutter-使用FlutterEntry的路由管理和参数传递
tags: [HarmonyOS，Flutter]
date: 2025-06-04 16:18:01
keywords: HarmonyOS,鸿蒙应用开发,跨平台,flutter,鸿蒙支持flutter,鸿蒙添加flutter模块,路由管理,参数传递
---

## 前言
我们在前面介绍了如何搭建环境，如何向现有鸿蒙工程中添加 flutter 模块，这篇文章中我们来看一下参数传递问题。
我们先看一下有哪些场景
1. native 打开 flutter 页面
2. flutter 打开 native 页面
3. flutter 打开 flutter 页面
4. native 返回 flutter 页面
5. flutter 返回 native 页面
6. flutter 返回 flutter 页面


一开始考虑的是 flutter 打开 flutter 的时候用 flutter 的路由，不需要 native 参与，比如 FlutterA 跳转到 FlutterB，直接在 flutter 进行跳转。但是也有可能会出现由 nativeA 打开 FlutterB，这时候 FlutterB 返回时需要将数据传给上个页面，如果按照一开始的逻辑，FlutterB 页面就需要判断来源，增加了复杂性。
因此，我们决定 flutter 打开 flutter 页需要重新打开一个新的 native 页面，让这个 native 页面加载对应的 flutter 页面。
也就是说，一个 native 页面只承载一个FlutterView。
这种情况下就需要我们写插件了。

## 创建插件
这里为了简单，我们在`my_flutter_module`下新建一个`plugins`文件，将插件工程放在这个文件夹下。
``` shell
cd my_flutter_module/plugins/
flutter create --org com.huangyuanlove.flutter_router --template=plugin --platforms=ohos flutter_router
```
这里我们演示鸿蒙项目下的插件，就没有支持 Android 和 iOS。
在 my_flutter_module中引用这个插件,在`pubspec.yaml`中添加引用
``` yaml
dependencies:
  flutter_router: 
    path: plugins/flutter_router
```


## 路由策略

1. flutter 页面路由统一调用插件中的方法进行跳转。
2. 在 flutter 的 main 函数中注册页面路径相关参数
3. 跳转时判断目标是 flutter 还是 native，如果是 flutter，则使用 Navigator；如果是 native ，则通过 methodChannel 调用
4. 




## 打开页面

### native打开 flutter 页面
指定路由路径后面添加参数

### flutter打开 native
指定路由，参数

### flutter打开 flutter
Navigation.push

## 关闭页面

### native 返回 flutter 页面

### flutter 返回 native 页面
### flutter 返回 flutter 页面