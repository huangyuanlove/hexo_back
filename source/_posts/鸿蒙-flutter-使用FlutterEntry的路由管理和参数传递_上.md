---
title: 鸿蒙-flutter-使用FlutterEntry的路由管理和参数传递_上
tags: [HarmonyOS,Flutter]
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

这里主要考虑flutter发起路由时的情况：
1. flutter 页面路由统一调用插件中的方法进行跳转。
2. 在 flutter 的 main 函数中注册页面路径相关参数
3. 跳转时判断目标是 flutter 还是 native，如果是 flutter，则使用 Navigator；如果是 native ，则通过 methodChannel 调用原生打开
4. 当页面返回时，先判断是否可以由flutter进行pop(flutter打开flutter页面的情况)，可以的话则由flutter进行pop，否则通过methodChannel调用原生进行返回

### native打开flutter

在前面的章节，我们了解了如何打开flutter页面，但仅仅是打开了默认的页面，并没有打开指定的页面和传入参数，在这一节中，我们来看下如何实现。



### flutter页面注册
由于我们确定是用路径+参数的形式来确定要打开的页面，因此需要我们先将路径和对应的flutter页面关联起来，并且能创建页面的时候把对应的参数传进去。
我们在flutter_router插件中创建一个用来记录路径和页面关系的类`router_manager.dart`,考虑到创建页面的时候需要传递参数，所以我们可以这么保存
``` dart
typedef RouteWidgetBuilder = Widget Function(dynamic args);
final Map<String, RouteWidgetBuilder> _routerMap = {};

bool addRouter(String path, RouteWidgetBuilder routerBuilder) {
  _routerMap[path] = routerBuilder;
  return true;
}
```

### 路径和参数获取
由于没有找到太好的方法来传递参数，暂时先将参数拼接到路径上，因此，我们还需要定义两个方法：从带参数路径中获取路径，从带参数路径中获取参数。
比如要打开的全路径为：`login?{"name":"harmonyos","age":3}`,我们获取到的路径为`login`,获取到的参数为`{"name":"harmonyos","age":3}`,也就是说路径和参数是用`?`拼接，并且参数是json字符串，这样方便我们解析。

``` dart
  String _getRouterName(String? path) {
    if (path == null) {
      debugPrint("获取路径出错，path 为空");
      return "/";
    }
    if (path.contains('?')) {
      var uri = Uri.parse(path);
      return uri.path;
    } else {
      return path;
    }
  }

  Object? _getRouteArgs(String? route) {
    if (route == null) {
      return null;
    }

    if (route.contains('?')) {
      var uri = Uri.parse(route);

      String query = uri.query;
      try {
        query = Uri.decodeFull(query);
        return json.decode(query);
      } catch (e) {
        // Map<String, String>
        return Uri.splitQueryString(query);
      }
    } else {
      return null;
    }
  }
```


我们还需要一个根据路径获取页面的方法
``` dart
Widget getRouterWidget(String path, {Object? params}) {
  final String routerName = _getRouterName(path);
  Object? pathParam = _getRouteArgs(path);
  debugPrint(
      "getRouterWidget path:$path, routerName:$routerName,pathParam:$pathParam ,params:$params");
  RouteWidgetBuilder? routerBuilder = _routerMap[routerName];
  if (routerBuilder != null) {
    return routerBuilder(pathParam ?? params);
  }
  return Container();
}
```
到这里我们就已经做好了native打开flutter页面的准备。

### 准备目标页面

这假设我们有一个login页面，
``` dart
//login.dart

import 'dart:convert';

import 'package:flutter/material.dart';

class LoginPage extends StatefulWidget {
  final Map<String, dynamic> args;

  const LoginPage(this.args, {super.key});
  @override
  State<LoginPage> createState() => _LoginPageState();
}

class _LoginPageState extends State<LoginPage> {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text("登录页面"),centerTitle: true,),
      body: Column(
        children: [
          const Text("登录页面获取到的参数",
              style: TextStyle(fontSize: 16, color: Color(0xff333333))),
          Text(jsonEncode(widget.args)),
          Text("name:${widget.args['name']}"),
          Text("age:${widget.args['age']}"),
        ],
      ),
    );
  }
}

```

### flutter中的处理
然后我们在flutter的main方法中注册，之后获取要打开的路径，获取对应页面和参数，最后展示出来.
我们在`main.dart`中做一下处理

``` dart
import 'dart:ui';

import 'package:flutter/material.dart';
import 'package:flutter_router/router_manager.dart';


import 'login_page.dart';

void main() {
  initRouter(); //注册flutter页面
  var routerName = PlatformDispatcher.instance.defaultRouteName;
  debugPrint("获取到需要加载的路径：${routerName}");
  runApp(MyApp(path: routerName));
}

class MyApp extends StatelessWidget {
  final String path;
  const MyApp({required this.path, super.key});

  // This widget is the root of your application.
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      home: RouterManager.instance.getRouterWidget(path),
    );

  }
}
void initRouter(){
  RouterManager.instance.addRouter("login",(args){return LoginPage(args);});
}
```

### 鸿蒙侧的处理
我们先从原生页面点击按钮之后跳转到flutter的登录页面，并且传递一些参数过去。
我这里使用HMRouter这个三方库来做的页面跳转。
添加flutter模块还是和前面的讲的一样，没啥太大的区别
#### EntryAbility
``` TypeScript
import UIAbility from '@ohos.app.ability.UIAbility';
import hilog from '@ohos.hilog';
import { AbilityConstant, Want } from '@kit.AbilityKit';
import { UIContext, window, uiObserver as observer } from '@kit.ArkUI';
import { JSON } from '@kit.ArkTS';
import { HMRouterMgr } from '@hadss/hmrouter';
import { ExclusiveAppComponent, FlutterManager } from '@ohos/flutter_ohos';

export default class EntryAbility extends UIAbility implements ExclusiveAppComponent<UIAbility>{

  detachFromFlutterEngine(): void {
  }
  getAppComponent(): UIAbility {
    return this;
  }
  private uiContext: UIContext | null = null;

  onCreate(want: Want, launchParam: AbilityConstant.LaunchParam): void {
    HMRouterMgr.init({
      context: this.context
    })
    FlutterManager.getInstance().pushUIAbility(this);
  }


  onDestroy(): void {
    FlutterManager.getInstance().popUIAbility(this);
  }

  onWindowStageCreate(windowStage: window.WindowStage): void {
    FlutterManager.getInstance().pushWindowStage(this, windowStage);

    windowStage.loadContent('pages/Index', (err) => {
      // windowStage.loadContent('pages/tel_inquiry_waiting_page/index', (err) => {
      if (err.code) {
        hilog.error(0x0000, 'testTag', 'Failed to load the content. Cause: %{public}s', JSON.stringify(err) ?? '');
      }
    });
  }

  onWindowStageDestroy(): void {
    FlutterManager.getInstance().popWindowStage(this);
    // Main window is destroyed, release UI related resources
    hilog.info(0x0000, 'testTag', '%{public}s', 'Ability onWindowStageDestroy');
    if (this.uiContext) {
      observer.off('routerPageUpdate', this.uiContext);
    }
  }

}

```
#### index.ets
主页

``` TypeScript
import { HMDefaultGlobalAnimator, HMNavigation, HMRouterMgr } from '@hadss/hmrouter';
import { AttributeUpdater } from '@kit.ArkUI';
import { FlutterMainPage } from './flutter/FlutterMainPage';

@Entry
@Component
struct Index {

  modifier: NavModifier = new NavModifier();
  build() {
    Column(){
      HMNavigation({
        navigationId: 'mainNavigation', options: {
          standardAnimator: HMDefaultGlobalAnimator.STANDARD_ANIMATOR,
          dialogAnimator: HMDefaultGlobalAnimator.DIALOG_ANIMATOR,
          modifier: this.modifier
        }
      }){
        Column() {

          FlutterMainPage()
        }
      }
    }
  }
}
class NavModifier extends AttributeUpdater<NavigationAttribute> {
  initializeModifier(instance: NavigationAttribute): void {
    instance.mode(NavigationMode.Stack);
    instance.navBarWidth('100%');
    instance.hideTitleBar(true);
    instance.hideToolBar(true);
  }
}
```

#### MyFlutterEntry
``` TypeScript

import { GeneratedPluginRegistrant } from '@ohos/flutter_module';
import { FlutterEngine, FlutterEntry } from '@ohos/flutter_ohos';


export class MyFlutterEntry  extends FlutterEntry {
  configureFlutterEngine(flutterEngine: FlutterEngine): void {
    super.configureFlutterEngine(flutterEngine)
    console.error("MyFlutterEntry configureFlutterEngine")
    GeneratedPluginRegistrant.registerWith(flutterEngine)
  }
}


```

#### MyFlutterPage

``` TypeScript
import { FlutterManager, FlutterPage, FlutterView } from '@ohos/flutter_ohos'
import { HMLifecycleState, HMRouter, HMRouterMgr } from '@hadss/hmrouter'
import { MyFlutterEntry } from './MyFlutterEntry'

@HMRouter({ pageUrl: 'pages/flutter/MyFlutterPage' })
@Component
export struct MyFlutterPage {
  private flutterEntry: MyFlutterEntry | null = null;
  private flutterView?: FlutterView

  aboutToAppear() {
    let param: Record<string, string> = HMRouterMgr.getCurrentParam() as Record<string, string>
    console.error(`MyFlutterPage params ${JSON.stringify(param)}`)
    this.flutterEntry = new MyFlutterEntry(getContext(this), param)
    this.flutterEntry.aboutToAppear()
    this.flutterView = this.flutterEntry.getFlutterView()
    HMRouterMgr.getCurrentLifecycleOwner()?.addObserver(HMLifecycleState.onShown, () => {
      this.flutterEntry?.onPageShow()
      FlutterManager.getInstance().setUseFullScreen(true)
    })
    HMRouterMgr.getCurrentLifecycleOwner()?.addObserver(HMLifecycleState.onHidden, () => {
      this.flutterEntry?.onPageHide()
      FlutterManager.getInstance().setUseFullScreen(false)
    })
    HMRouterMgr.getCurrentLifecycleOwner()?.addObserver(HMLifecycleState.onBackPressed, (): boolean => {
      // this.flutterEntry?.onBackPress()
      // (getContext(this) as common.UIAbilityContext).eventHub.emit('EVENT_BACK_PRESS')
      HMRouterMgr.pop()
      return true
    })
  }

  aboutToDisappear() {
    this.flutterEntry?.aboutToDisappear()
  }

  build() {

      FlutterPage({ viewId: this.flutterView?.getId() }).expandSafeArea([SafeAreaType.SYSTEM], [SafeAreaEdge.BOTTOM])

  }
}
```
#### FlutterMainPage

``` TypeScript
import { HMPopInfo, HMRouter } from '@hadss/hmrouter';
import { ActionBar } from '../../comm/ActionBar';
import { FlutterPath, MyFlutterRouter } from './MyFlutterRouter';

@Component
export struct FlutterMainPage {
  @State message: string = '';
  @State popResult:Map<string,Object> | undefined = undefined
  build() {
    Column() {
      ActionBar({title:"flutter测试页面"})
      Button("HMRouter flutter").onClick((_) => {
        MyFlutterRouter.push(FlutterPath.LOGIN, { "name": "harmonyos", "age": 3 }, {
          onResult: (popInfo: HMPopInfo) => {
            console.error(`FlutterMainPage.push result ${popInfo.result}`)
            this.popResult = popInfo.result as Map<string,Object>
            if(this.popResult){
              this.message = ""
              this.popResult.forEach((value,key)=>{
                this.message += `${key} : ${value} \n`
              })
            }
          },
          onArrival: () => {
            console.error(`FlutterMainPage.push onArrival `)
          },
          onLost: () => {
            console.error(`FlutterMainPage.push onLost `)
          }
        })
      })

      Text("页面返回携带的参数")
      Text(this.message)

    }
    .height('100%')
    .width('100%')
  }
}
```

#### MyFlutterRouter
``` TypeScript
import { HMPopInfo, HMRouterMgr, HMRouterPathCallback } from '@hadss/hmrouter'
import { JSON } from '@kit.ArkTS'
import { FlutterAbilityLaunchConfigs } from '@ohos/flutter_ohos'

export enum FlutterPath {
  MAIN = 'main',
  LOGIN = 'login',
  TEST = 'test',
  ROOT = '/'
}


export class MyFlutterRouter {
  static push(path: FlutterPath, params: Record<string, Object> | undefined = undefined,
    callback?: HMRouterPathCallback) {
    let target: string = path.toString()
    if (params) {
      target = `${target}?${JSON.stringify(params)}`
    }
    let routerParams: Record<string, string> = {}
    routerParams[FlutterAbilityLaunchConfigs.EXTRA_INITIAL_ROUTE] = target;

    HMRouterMgr.push({ pageUrl: 'pages/flutter/MyFlutterPage', param: routerParams }, {
      onResult: (popInfo: HMPopInfo) => {
        console.error(`获取到页面返回时携带的参数 ${JSON.stringify(popInfo.result)}`)
        if (callback && callback.onResult) {
          callback.onResult(popInfo)
        }
      },
      onArrival: () => {
        if (callback && callback.onArrival) {
          callback.onArrival()
        }
      },
      onLost: () => {
        if (callback && callback.onLost) {
          callback.onLost()
        }
      }
    })
  }
}
```
## 效果
这样我们就完成了从native侧跳转到指定页面，并且还可以将参数传递过去。
这里需要特别注意的是在`MyFlutterRouter`中，
``` TypeScript
let routerParams: Record<string, string> = {}
routerParams[FlutterAbilityLaunchConfigs.EXTRA_INITIAL_ROUTE] = target;
```
这里的参数key用的是`FlutterAbilityLaunchConfigs.EXTRA_INITIAL_ROUTE`,值为`route`。
这是因为在`FlutterEntry`中`getInitialRoute`方法中，获取初始路径使用的就是这个
``` TypeScript
  getInitialRoute(): string {
    if (this.parameters![FlutterAbilityLaunchConfigs.EXTRA_INITIAL_ROUTE]) {
      return this.parameters![FlutterAbilityLaunchConfigs.EXTRA_INITIAL_ROUTE] as string
    }
    return "";
  }
```
看下效果图：

![鸿蒙打开flutter指定页面并且传递参数](image/harmony_flutter/harmony_flutter_initial_route.gif)