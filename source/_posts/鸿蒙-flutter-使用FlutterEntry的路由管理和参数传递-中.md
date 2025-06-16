---
title: 鸿蒙-flutter-使用FlutterEntry的路由管理和参数传递_中
tags: [HarmonyOS,Flutter]
date: 2025-06-08 16:06:54
keywords: HarmonyOS,鸿蒙应用开发,跨平台,flutter,鸿蒙支持flutter,鸿蒙添加flutter模块,路由管理,参数传递
---

## 前言
前面我们完成了鸿蒙打开flutter指定页面，并且传递参数，接下来我们看一下在flutter侧打开鸿蒙原生页面，并且传递参数应该如何处理。
当然了，我们在前面也提到了，在flutter发起路由的时候，都交给插件来处理。并且我们在上一章中也创建好了flutter插件，并没有使用和原生交互，只是创建了一个flutter路由和页面映射的管理类。


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



## flutter打开native

这里需要用到和native通信了。前面也提到过，当flutter发起路由时，先判断目标页面是不是flutter页面，是的话用flutter中的Navigator打开，否则调用channel通知原生打开。

因为打开原生页面有时候也需要传递一些参数，我们在`FlutterRouterPlatform`中定义这么一个方法：
``` dart
Future<T> open<T extends Object?>(url, {dynamic arguments}) {
  throw UnimplementedError('open() has not been implemented.');
}

```
然后我们在`MethodChannelFlutterRouter`中实现这个方法：
``` dart
  @override
  Future<T> open<T extends Object?>(url,
      {dynamic arguments}) async {
    var args = {};
    args['path'] = url;

    if (arguments != null) {
      args['arguments'] = arguments;
    }
    debugPrint("-----------open---start--------");
    debugPrint("path $url");
    debugPrint("arguments $arguments");
    debugPrint("-----------open----end-------");

    final result = await methodChannel.invokeMethod('open', args);
    return result;
  }
```
看着代码挺多，实际上只是把`url`和`arguments`这两个参数打包到了`args`里面，通过`methodChannel`传给原生。


### 原生侧处理
这里我们可以使用DevEco打开插件目录下的ohos文件夹，把它当作一个鸿蒙工程。或者简单点，直接在当前工程中编辑也行。只是方法提示不太友好，我们可以把ohos工程中的`FlutterRouterPlugin`文件直接复制到当前鸿蒙工程中，编辑完后再复制回去。
我们看下`FlutterRouterPlugin`应该如何处理。
考虑到我们是传入的路径，也是打算注册路径和关联页面，但考虑到我们的实际业务情况，就开放了一个处理接口，由native侧设置，当触发打开native页面时，由native来处理

``` TypeScript
import {
  FlutterPlugin,
  FlutterPluginBinding,
  MethodCall,
  MethodCallHandler,
  MethodChannel,
  MethodResult,
} from '@ohos/flutter_ohos';

/** FlutterRouterPlugin **/
export default class FlutterRouterPlugin implements FlutterPlugin, MethodCallHandler {
  private channel: MethodChannel | null = null;
  static routerPushHandler: (path: string, args: Record<string, Object> | undefined,result:MethodResult) => boolean = (path, args,result) => {
    return false
  };

  static setRouterPushHandler(handler: (path: string, args: Record<string, Object> | undefined,result:MethodResult) => boolean) {
    FlutterRouterPlugin.routerPushHandler = handler
  }

  constructor() {
  }

  getUniqueClassName(): string {
    return "FlutterRouterPlugin"
  }

  onAttachedToEngine(binding: FlutterPluginBinding): void {
    this.channel = new MethodChannel(binding.getBinaryMessenger(), "flutter_router");
    this.channel.setMethodCallHandler(this)
  }

  onDetachedFromEngine(binding: FlutterPluginBinding): void {
    if (this.channel != null) {
      this.channel.setMethodCallHandler(null)
    }
  }

  onMethodCall(call: MethodCall, result: MethodResult): void {
    if (call.method == "getPlatformVersion") {
      result.success("OpenHarmony ^ ^ ")
    } else if (call.method == 'open') {
      let path: string = call.argument('path')
      let args: Record<string, Object> | undefined = call.argument('arguments')
      console.error("-------onMethodCall----open---start--------")
      console.error(`path ${path}`)
      console.error(`arguments ${args}`)
      console.error("------onMethodCall-----open----end-------")
      FlutterRouterPlugin.routerPushHandler(path, args,result)

    } else {
      result.notImplemented()
    }
  }
}
```
我们再写一个鸿蒙页面，用来测试flutter打开native的情况
``` TypeScript
import { HMRouter, HMRouterMgr } from "@hadss/hmrouter";
import { ActionBar } from "../../comm/ActionBar";
import { UIUtils } from "@kit.ArkUI";

@HMRouter({ pageUrl: 'pages/flutter/FromFlutterPage' })
@Component
export struct FromFlutterPage {
  @State routerParam: Map<string, Object> | undefined = undefined

  aboutToAppear(): void {
    this.routerParam = HMRouterMgr.getCurrentParam() as Map<string, Object>
  }

  build() {
    Column() {
      ActionBar({ title: "从 flutter 打开的页面",onClickBack:(_)=>{HMRouterMgr.pop()} })

      Text('获取到的路由参数')
      Text(this.routerParamsToStr())

    }
  }

  routerParamsToStr(): string {
    if (this.routerParam) {
      let result = ''

      let tmp:Map<string,Object> = UIUtils.getTarget<Map<string,Object>>(this.routerParam);
      tmp.forEach((value,key)=>{
        result += `${key} : ${value} \n`
      })

      return result

    } else {
      return "无参数"
    }
  }
}
```
然后我们在`EntryAbility`的`onCreate`方法中设置一下`routerPushHandler`
``` TypeScript
    FlutterRouterPlugin.setRouterPushHandler((path:string,args:Record<string,Object>|undefined,result: MethodResult)=>{
      console.error(`rouerHandler:path=> ${path} ,args=>${args}`)
      if(path =='pages/flutter/FromFlutterPage'){
        HMRouterMgr.push({pageUrl:'pages/flutter/FromFlutterPage',param:args})
        return true
      }
      return false;
    });
```
这里我们判断了path的值和跳转对应的页面。

### flutter侧调用
我们调用的方法都写在了`FlutterRouter`中,并且也是在这里判断是打开 flutter 页面还是打开native 页面
我们在`RouterManager`中添加一个判断是否为 flutter 页面的方法
``` dart
bool hasRouterWidget(String path) {
  final String routerName = _getRouterName(path);
  return _routerMap.containsKey(routerName);
}
```
然后我们在`FlutterRouter`写一下对应的打开页面的方法:
``` dart


import 'package:flutter/material.dart';
import 'package:flutter_router/router_manager.dart';

import 'flutter_router_platform_interface.dart';

class FlutterRouter {
  Future<String?> getPlatformVersion() {
    return FlutterRouterPlatform.instance.getPlatformVersion();
  }
  Future<T?> open<T extends Object?>(context, path,
      {Object? arguments}) async {
    final bool useFlutterPage = RouterManager.instance.hasRouterWidget(path);
    if(useFlutterPage){
      debugPrint("FlutterRouter#open 打开 flutter");
      Widget target =  RouterManager.instance.getRouterWidget(path,params: arguments);
      return Navigator.of(context).push(MaterialPageRoute(
        builder: (context) => target,
      ),);
      // return Navigator.of(context).pushNamed(path, arguments: arguments);
    } else {
      // 打开native页面: path已在native端注册
      debugPrint("FlutterRouter#open 打开 native");
      return FlutterRouterPlatform.instance
          .open<T>(path, arguments: arguments)
          .then((value) {
        return value;
      });
    }
  }
}

```
之后我们在之前使用的`LoginPage`页面中调用一下这个方法:
``` dart
ElevatedButton(
  onPressed: () {
    //HMRouterAPage
    FlutterRouter().open(context, 'pages/flutter/FromFlutterPage',
        arguments: {
          'name': 'flutter_harmony',
          'age': 3
        });
  },
  child: const Text("FromFlutterPage",
      style: TextStyle(fontSize: 16, color: Color(0xff333333))),
),
```

看一下效果
![](image/harmony_flutter/flutter_to_native.gif)

## 总结一下

1. flutter打开页面时，调用`FlutterRouter#open`方法，
2. 在该方法中判断目标页面是 flutter 页面还是 native 页面
3. 如果是 native 页面，则通过 methodChannel 调用 native 方法
4. 最终调用的 native 方法由宿主原生工程中设置，由宿主原生工程来打开对应页面



