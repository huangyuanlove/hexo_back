---
title: 鸿蒙-flutter-使用FlutterEntry的路由管理和参数传递_下
tags: [HarmonyOS，Flutter]
date: 2025-06-10 11:44:44
keywords: HarmonyOS,鸿蒙应用开发,跨平台,flutter,鸿蒙支持flutter,鸿蒙添加flutter模块,路由管理,参数传递
---

## 前言
前面两篇介绍了如何打开指定页面，并且传递对应的参数。这一篇中我们来看下在页面返回时，如何将数据传递给上一个页面。

## 方案
按照之前的介绍，我们在`flutter`打开`native`时，也是通过`methodChannel`调用原生的方法打开的对应页面，那么当原生页面返回的时候，也是会将数据放在`HMRouterPathCallback`回调中返回。当我们获取到对应的数据之后，可以通过`MethodResult`将数据返回给`flutter`。
当`flutter`页面返回时，需要先判断能不能由`flutter`进行返回，不能返回的话再通过`methodChannel`调用 `native`返回。

## 实现

### native页面返回
对于 native 页面返回，我们看一下上一篇中提到的 flutter 打开 native，在·EntryAbility·中调用的`setRouterPushHandler`，当页面返回的时候，会回调`HMRouterMgr.push`传入的`HMRouterPathCallback`对象中的方法。我们来改造一下`setRouterPushHandler`这个方法，将 MethodResult 对象也传过来，当页面返回的时候调用MethodResult.success将参数返回。

``` TypeScript
FlutterRouterPlugin.setRouterPushHandler((path:string,args:Record<string,Object>|undefined,result: MethodResult)=>{
  console.error(`rouerHandler:path=> ${path} ,args=>${args}`)
  if(path =='pages/flutter/FromFlutterPage'){
    HMRouterMgr.push({pageUrl:'pages/flutter/FromFlutterPage',param:args},{
      onResult:(popInfo:PopInfo)=>{
        if(popInfo){
          result.success(popInfo.result)
        }else{
          result.success("")
        }
      }
    })
    return true
  }
  return false;
});
```

### flutter页面返回
按照之前的介绍，flutter 返回也统一调用插件的返回方法。
在插件的`FlutterRouterPlatform`中添加`pop`方法:
``` dart
void pop([args]) {
  throw UnimplementedError('open() has not been implemented.');
}
```
在插件的`MethodChannelFlutterRouter`中添加实现：
``` dart
@override
void pop([args]) async {
  await methodChannel.invokeMethod('pop', args);
}
```
在插件的`FlutterRouterPlugin`中添加调用，这里我们还是将返回的处理交给宿主项目决定
``` TypeScript
static routerPopHandler:(call:MethodCall,result: MethodResult)=>boolean =(call,result)=>{
  return false
}
static setRouterPopHandler(handler:(call:MethodCall,result: MethodResult)=>boolean){
  FlutterRouterPlugin.routerPopHandler = handler
}
onMethodCall(call: MethodCall, result: MethodResult): void {
  if(call.method =='pop'){
    FlutterRouterPlugin.routerPopHandler(call,result)
  }
}
```
在插件的`FlutterRouter`中统一调用返回的方法：先判断 flutter 是否可以 pop
``` dart
void pop(context, [dynamic args]) {
  if (Navigator.of(context).canPop()) {
    Navigator.of(context).pop(args);
  } else {
    FlutterRouterPlatform.instance.pop(args);
  }
}
```
为了测试简单全面，我们在 flutter_module中添加一个 flutter 页面。
``` dart
//from_flutter_page.dart

import 'package:flutter/material.dart';
import 'package:flutter_router/flutter_router.dart';
class FromFlutterPage extends StatefulWidget {
  final Map<String, dynamic> args;
  const FromFlutterPage(this.args,{  super.key});

  @override
  State<FromFlutterPage> createState() => _FromFlutterPageState();
}

class _FromFlutterPageState extends State<FromFlutterPage> {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(centerTitle: true,title: Text('from flutter',style: TextStyle(fontSize: 16,color: Color(0xff333333)),),),
      body: Column(
        children: [
            Text("收到上个页面传过来的数据"),
          Text("${widget.args}"),
          ElevatedButton(
            onPressed: () {
              FlutterRouter().pop(context,{"user_id":"xuan","page_name":"FromFlutterPage"});
            },
            child: const Text("带数据返回上个页面",
                style: TextStyle(fontSize: 16, color: Color(0xff333333))),
          ),
        ],
      ),
    );
  }
}

```
注意，添加的页面需要在 flutter 侧注册一下
``` dart
//main.dart
void initRouter(){
  RouterManager.instance.addRouter("login",(args){return LoginPage(args);});
  RouterManager.instance.addRouter("from_flutter",(args){return FromFlutterPage(args);});
}
```
之后我们在LoginPage中添加一下相关跳转、返回、参数展示
``` dart
//login_page.dart
import 'dart:convert';

import 'package:flutter/material.dart';
import 'package:flutter_router/flutter_router.dart';

class LoginPage extends StatefulWidget {
  final Map<String, dynamic> args;

  const LoginPage(this.args, {super.key});

  @override
  State<LoginPage> createState() => _LoginPageState();
}

class _LoginPageState extends State<LoginPage> {
  String nativeResult = "";
  String flutterResult = "";

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text("登录页面"),
        centerTitle: true,
      ),
      body: Column(
        children: [
          const Text("登录页面获取到的参数",
              style: TextStyle(fontSize: 16, color: Color(0xff333333))),
          Text(jsonEncode(widget.args)),
          Text("name:${widget.args['name']}"),
          Text("age:${widget.args['age']}"),
          ElevatedButton(
            onPressed: () {
              //HMRouterAPage
              FlutterRouter().open(context, 'pages/flutter/FromFlutterPage',
                  arguments: {
                    'name': 'flutter_harmony',
                    'age': 3
                  }).then((value) {
                debugPrint("native页面返回 flutter 传递的参数 ${jsonEncode(value)}");
                setState(() {
                  nativeResult = jsonEncode(value);
                });
              });
            },
            child: const Text("FromFlutterPage",
                style: TextStyle(fontSize: 16, color: Color(0xff333333))),
          ),
          ElevatedButton(
            onPressed: () {
              FlutterRouter().open(context, "from_flutter", arguments: {
                "from": "LoginPage",
                "business_id": "123"
              }).then((value) {
                debugPrint("flutter页面返回 flutter 传递的参数 ${jsonEncode(value)}");
                setState(() {
                  flutterResult = jsonEncode(value);
                });
              });
            },
            child: const Text("跳转到 flutter",
                style: TextStyle(fontSize: 16, color: Color(0xff333333))),
          ),
          ElevatedButton(
            onPressed: () {
              FlutterRouter().pop(context, {"user_id": "xuan"});
            },
            child: const Text("返回上个页面携带参数",
                style: TextStyle(fontSize: 16, color: Color(0xff333333))),
          ),
          Container(
            margin: const EdgeInsets.all(15),
            color: const Color(0xff9c649a),
            child: Column(
              children: [const Text('flutter页面返回携带的参数：'), Text(flutterResult)],
            ),
          ),
          Container(
            margin: const EdgeInsets.all(15),
            color: const Color(0xff7b7a32),
            child: Column(
              children: [const Text('native页面返回携带的参数：'), Text(nativeResult)],
            ),
          ),
        ],
      ),
    );
  }
}

```
这样我们在 flutter 中需要做的事情就已经做完了，执行`flutter build har --debug`会在`my_flutter_module/.ohos/har`文件夹下看到三个**har**文件，直接复制到鸿蒙工程中进行依赖就行了。记得把源码依赖删掉
``` json
//oh-package.json5
{
  "modelVersion": "5.0.5",
  "description": "Please describe the basic information.",
  "dependencies": {
    "@hadss/hmrouter": "1.0.0-rc.10",
    "@ohos/flutter_ohos": "file:./har/flutter.har",
    "@ohos/flutter_module": ".//har/flutter_module.har",
    "flutter_router": "file:./har/flutter_router.har"
  }
  "overrides": {
    "@ohos/flutter_ohos": "file:./har/flutter.har",
    "flutter_router": "file:./har/flutter_router.har"
  }
}

```
最后一步，在`EntryAbility`中设置一下处理返回的调用
``` TypeScript
onCreate(want: Want, launchParam: AbilityConstant.LaunchParam): void {
  FlutterRouterPlugin.setRouterPopHandler((call:MethodCall,result: MethodResult)=>{
    console.error(`flutter 调用 pop  setRouterPopHandler:  ${call.method}  ${call.args}`)
    HMRouterMgr.pop({param:call.args})
    return true;
  })
}
```
这样我们就完成了页面返回时参数的传递。

## 结论
先看下效果
![](image/harmony_flutter/go_back_params.gif)

总结一下：
1. native 打开 flutter 时，将参数拼接到初始化路径上。
2. flutter 侧需要将所有的页面注册到插件中
3. flutter 打开页面时，插件判断目标页面是 flutter 还是 native，选择对应的方法打开页面
4. 页面返回时，通过 MethodResult 将数据返回

----
至此，鸿蒙和 flutter 混编的介绍就已经结束了。