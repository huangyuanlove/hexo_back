---
title: 鸿蒙-flutter-如何向现有鸿蒙项目中添加flutter模块
tags: [HarmonyOS,Flutter]
date: 2025-04-29 10:37:47
keywords: HarmonyOS,鸿蒙应用开发,跨平台,flutter,鸿蒙支持flutter,鸿蒙添加flutter模块
---

## 前言
在版本迭代时，有部分新增的功能，需要开发新的业务模块，这里希望使用跨平台框架，实现代码Android、iOS、HarmonyOS 复用。之前的部分业务使用flutter 开发，HarmonyOS 也支持 flutter 的接入，因此，这次依然使用选择使用 flutter。

## 环境

在上一篇中详细介绍了如何安装和配置环境，flutter使用的是`3.22.0-ohos`的版本，如果需要使用多个flutter版本的话，可以使用fvm来管理和切换多个版本

## 集成

### 创建 flutter 模块
在鸿蒙项目文件夹中创建flutter module

``` shell
# 为方便代码管理，将flutter代码放到工程目录内统一管理
cd HarmonyProject
# 1.以module形式集成到项目，创建flutter_module
flutter create -t module my_flutter_module
# 2.构建har文件,注意，这里需要限制性一下flutter的构建，否则是没有har文件的
cd flutter_module
flutter build har --release
``` 
### 引用 flutter

这里有两种方案，直接应用源码和引用har文件。
我们可以在开发阶段引用源码，在测试发版时引用 har 文件。
#### 引用源码


修改鸿蒙工程根目录下的`oh-package.json5`文件并添加对应的依赖
``` json5
"dependencies":{
    "@ohos/flutter_ohos": "file:./my_flutter_module/.ohos/har/flutter.har",
    "@ohos/flutter_module": "./my_flutter_module/.ohos/flutter_module"
}
```
修改项目工程下的`build-profile.json5`文件，添加一个新的module
``` json
// 以下为新增内容
{
    "name": "flutter_module",
    "srcPath": "./my_flutter_module/.ohos/flutter_module",
    "targets": [
    {
        "name": "default",
        "applyToProducts": [
        "default"
        ]
    }
    ]
}
```

#### 引用har文件

在测试发版的时候先将 flutter 模块打成 har 包，复制到鸿蒙项目中，直接引用 har 文件就好。

``` shell
# 2.构建har文件
cd my_flutter_module
flutter build har --release
```
我们可以在`my_flutter_module/.ohos/har`文件夹下看到两个har文件： `flutter.har`和`flutter_module.har`。
我们可以将这两个文件复制到harmony项目中直接应用。

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
需要注意的是，在切换引用方式的时候，记得修改工程下的`build-profile.json5`文件。

## 加载

我们这里使用FlutterEntry来展示flutter相关页面。

### EntryAbility 可以继承 UIAbility

``` TypeScript
export default class EntryAbility extends UIAbility implements ExclusiveAppComponent<UIAbility> {

  detachFromFlutterEngine(): void {
    // throw new Error('Method not implemented.');
  }

  getAppComponent(): UIAbility {
    return this;
  }

  onCreate(want: Want, launchParam: AbilityConstant.LaunchParam): void {
    FlutterManager.getInstance().pushUIAbility(this);
  }

  onDestroy(): void | Promise<void> {
    FlutterManager.getInstance().popUIAbility(this);
  }

  onWindowStageCreate(windowStage: window.WindowStage): void {
    windowStage.getMainWindowSync().setWindowLayoutFullScreen(true);
    FlutterManager.getInstance().pushWindowStage(this, windowStage);
    windowStage.loadContent('pages/Index');
  }

  onWindowStageDestroy() {
    FlutterManager.getInstance().popWindowStage(this);
  }
}
```

###  继承 FlutterEntry 并注册插件


``` TypeScript
export default class MyFlutterEntry extends FlutterEntry {
  configureFlutterEngine(flutterEngine: FlutterEngine): void {
    super.configureFlutterEngine(flutterEngine);
    GeneratedPluginRegistrant.registerWith(flutterEngine);
    this.delegate?.addPlugin(new BatteryPlugin());
  }
}
```

### FlutterEntry 需要和 FlutterView 一起使用

``` TypeScript
@Entry
@Component
struct Index {
  private context = getContext(this) as common.UIAbilityContext
  private flutterEntry: FlutterEntry | null = null;
  private flutterView?: FlutterView

  aboutToAppear() {
    Log.d("Flutter", "Index aboutToAppear===");
    this.flutterEntry = new MyFlutterEntry(getContext(this))
    this.flutterEntry.aboutToAppear()
    this.flutterView = this.flutterEntry.getFlutterView()
  }

  aboutToDisappear() {
    Log.d("Flutter", "Index aboutToDisappear===");
    this.flutterEntry?.aboutToDisappear()
  }

  // Navigation的生命周期是onShown
  onPageShow() {
    Log.d("Flutter", "Index onPageShow===");
     FlutterManager.getInstance().setUseFullScreen(true)
    this.flutterEntry?.onPageShow()
  }

  // Navigation的生命周期是onHidden
  onPageHide() {
    Log.d("Flutter", "Index onPageHide===");
     FlutterManager.getInstance().setUseFullScreen(false)
    this.flutterEntry?.onPageHide()
  }

  build() {
    Stack() {
      FlutterPage({ viewId: this.flutterView?.getId() })
  }

  onBackPress(): boolean {
    this.context.eventHub.emit('EVENT_BACK_PRESS')
    return true
  }
}
```

## 效果
运行后我们使用router跳转到这个页面，发现是可以加载出来的，点击页面按钮行为也是正常的。

![flutter entry示例](image/harmony_flutter/flutter_entry_demo.gif)

## 后续

接下来我们继续看下路由管理以及参数传递问题