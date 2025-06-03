---
title: 鸿蒙-flutter-使用PlatformView
tags: [HarmonyOS，Flutter]
date: 2025-06-10 16:14:54
keywords:
---

我们自己的业务比较简单，基本上没有使用PlatformView，所有的页面要么是原生，要么是flutter，没有这种在flutter页面上展示原生控件的需求。
这里介绍一下如何在纯flutter项目中使用platformView展示鸿蒙组件。

## 准备
按照之前的环境搭建和第一个helloworld，搭建好环境，运行起来。

## 原生侧
使用DevEco打开项目工程下的ohos文件夹，DevEco会将该文件夹识别为一个鸿蒙项目，可以获得完整的代码提示和语法高亮。
我们先从底层向接口方向编写代码。

### 需要展示的View
定义一个用来在Flutter上展示的 Component。

``` TypeScript
import { CustomView } from "./CustomView" //这里的CustomView是我们后面需要继承PlatformView的类
import { Params } from '@ohos/flutter_ohos/src/main/ets/plugin/platform/PlatformView';
@Component
export struct ButtonComponent {
  @Prop params: Params
  customView: CustomView = this.params.platformView as CustomView
  @StorageLink('numValue') storageLink: string = "first"
  @State bkColor: Color = Color.Red

  build() {
    Column() {
      Button("发送数据给Flutter")
        .border({ width: 2, color: Color.Blue})
        .backgroundColor(this.bkColor)
        .onTouch((event: TouchEvent) => {
          console.log("nodeController button on touched")
        })
        .onClick((event: ClickEvent) => {
          this.customView.sendMessage();
          console.log("nodeController button on click")
        })

      Text(`来自Flutter的数据 : ${this.storageLink}`)
        .onTouch((event: TouchEvent) => {
          console.log("nodeController text on touched")
        })

    }.alignItems(HorizontalAlign.Center)
    .justifyContent(FlexAlign.Center)
    .direction(Direction.Ltr)
    .width('100%')
    .height('100%')
  }
}

```

### PlatformView的编写
我们需要继承`PlatformView`,并且实现其中的`getView`方法，返回一个`WrappedBuilder`, 在这个`WrappedBuilder`中，返回我们上面自定义的`ButtonComponent`。
当然免不了互相传输数据，因此我们这里还需要实现`MethodCallHandler`接口.
``` TypeScript
import MethodChannel, {
  MethodCallHandler,
  MethodResult
} from '@ohos/flutter_ohos/src/main/ets/plugin/common/MethodChannel';
import PlatformView, { Params } from '@ohos/flutter_ohos/src/main/ets/plugin/platform/PlatformView';
import common from '@ohos.app.ability.common';
import { BinaryMessenger } from '@ohos/flutter_ohos/src/main/ets/plugin/common/BinaryMessenger';
import StandardMethodCodec from '@ohos/flutter_ohos/src/main/ets/plugin/common/StandardMethodCodec';
import MethodCall from '@ohos/flutter_ohos/src/main/ets/plugin/common/MethodCall';
import { ButtonComponent } from './ButtonComponent';

@Observed
export class CustomView extends PlatformView implements MethodCallHandler {
  numValue: string = "test";

  methodChannel: MethodChannel;
  index: number = 1;

  constructor(context: common.Context, viewId: number, args: ESObject, message: BinaryMessenger) {
    super();
    console.log("nodeController viewId:" + viewId)
    // 注册消息通道，消息通道根据具体需求添加，代码仅作为示例
    this.methodChannel = new MethodChannel(message, `com.huangyuanlove/customView${viewId}`, StandardMethodCodec.INSTANCE);
    this.methodChannel.setMethodCallHandler(this);
  }

  onMethodCall(call: MethodCall, result: MethodResult): void {
    // 接受Dart侧发来的消息
    let method: string = call.method;
    let link1: SubscribedAbstractProperty<number> = AppStorage.link('numValue');
    switch (method) {
      case 'getMessageFromFlutterView':
        let value: ESObject = call.args;
        this.numValue = value;
        link1.set(value)
        console.log("nodeController receive message from dart: " + this.numValue);
        result.success(true);
        break;
    }
  }

  public sendMessage = () => {
    console.log("nodeController sendMessage")
    //向Dart侧发送消息
    this.methodChannel.invokeMethod('getMessageFromOhosView', 'natvie - ' + this.index++);
  }

  getView(): WrappedBuilder<[Params]> {
    return new WrappedBuilder(ButtonBuilder);
  }

  dispose(): void {
  }
}
@Builder
export function ButtonBuilder(params: Params) {
  ButtonComponent({ params: params })
    .backgroundColor(Color.Yellow)
}
```

### 自定义PlatformViewFactory

在这里需要在其create方法中创建自定义的PlatformView的实例。这个`PlatformViewFactory`主要就干这件事情。

``` TypeScript
import common from '@ohos.app.ability.common';
import MessageCodec from '@ohos/flutter_ohos/src/main/ets/plugin/common/MessageCodec';
import PlatformViewFactory from '@ohos/flutter_ohos/src/main/ets/plugin/platform/PlatformViewFactory';
import { BinaryMessenger } from '@ohos/flutter_ohos/src/main/ets/plugin/common/BinaryMessenger';
import PlatformView from '@ohos/flutter_ohos/src/main/ets/plugin/platform/PlatformView';
import { CustomView } from './CustomView';

export class CustomFactory extends PlatformViewFactory {
  message: BinaryMessenger;

  constructor(message: BinaryMessenger, createArgsCodes: MessageCodec<Object>) {
    super(createArgsCodes);
    this.message = message;
  }

  public create(context: common.Context, viewId: number, args: Object): PlatformView {
    return new CustomView(context, viewId, args, this.message);
  }
}
```

### 自定义FlutterPlugin

这里我们需要自定义一个继承于FlutterPlugin的CustomPlugin插件，在onAttachedToEngine中，注册自定义的PlatformViewFactory。
``` TypeScript
import  { FlutterPlugin,
  FlutterPluginBinding } from '@ohos/flutter_ohos/src/main/ets/embedding/engine/plugins/FlutterPlugin';
import StandardMessageCodec from '@ohos/flutter_ohos/src/main/ets/plugin/common/StandardMessageCodec';
import { CustomFactory } from './CustomFactory';

export class CustomPlugin implements FlutterPlugin {
  getUniqueClassName(): string {
    return 'CustomPlugin';
  }

  onAttachedToEngine(binding: FlutterPluginBinding): void {
    binding.getPlatformViewRegistry()?.
    registerViewFactory('com.huangyuanlove/customView', new CustomFactory(binding.getBinaryMessenger(), StandardMessageCodec.INSTANCE));
  }

  onDetachedFromEngine(binding: FlutterPluginBinding): void {}
}
```

### 添加Plugin
现在我们需要将上面自定义的plugin在`EntryAbility`中注册一下.
``` TypeScript

import { FlutterAbility, FlutterEngine } from '@ohos/flutter_ohos';
import { GeneratedPluginRegistrant } from '../plugins/GeneratedPluginRegistrant';
import { CustomPlugin } from '../widget/CustomPlugin';

export default class EntryAbility extends FlutterAbility {
  configureFlutterEngine(flutterEngine: FlutterEngine) {
    super.configureFlutterEngine(flutterEngine)
    GeneratedPluginRegistrant.registerWith(flutterEngine)
    this.addPlugin(new CustomPlugin());
  }
}

```

  至此，我们完成了原生侧的开发，下面看一下flutter侧怎么搞

## Flutter侧

### 用于发送和接收数据的Controller
这里我们先封装一个用于和原生侧进行数据交互的类，就叫`CustomViewController`了。
``` dart
class CustomViewController {
  final MethodChannel _channel;
  final StreamController<String> _controller = StreamController<String>();

  CustomViewController._(
    this._channel,
  ) {
    _channel.setMethodCallHandler(
      (call) async {
        switch (call.method) {
          case 'getMessageFromOhosView':
            // 从native端获取数据
            final result = call.arguments as String;
            _controller.sink.add(result);
            break;
        }
      },
    );
  }

  Stream<String> get customDataStream => _controller.stream;

  // 发送数据给native
  Future<void> sendMessageToOhosView(String message) async {
    await _channel.invokeMethod(
      'getMessageFromFlutterView',
      message,
    );
  }
}

```
这个类不封装也行，看自己的喜好。

### 用于展示原生控件的Widget
flutter侧比较简单，只需要搞一个用来展示原生控件的Widget就可以了，交互的话还是走channel，就是用上面封装的`CustomViewController`.

``` dart
import 'dart:async';

import 'package:flutter/material.dart';
import 'package:flutter/services.dart';

typedef OnViewCreated = Function(CustomViewController);

///自定义OhosView
class CustomOhosView extends StatefulWidget {
  final OnViewCreated onViewCreated;

  const CustomOhosView(this.onViewCreated, {Key? key}) : super(key: key);

  @override
  State<CustomOhosView> createState() => _CustomOhosViewState();
}

class _CustomOhosViewState extends State<CustomOhosView> {
  late MethodChannel _channel;

  @override
  Widget build(BuildContext context) {
    return _getPlatformFaceView();
  }

  Widget _getPlatformFaceView() {
    return OhosView(
      viewType: 'com.huangyuanlove/customView',
      onPlatformViewCreated: _onPlatformViewCreated,
      creationParams: const <String, dynamic>{'initParams': 'hello world'},
      creationParamsCodec: const StandardMessageCodec(),
    );
  }

  void _onPlatformViewCreated(int id) {
    _channel = MethodChannel('com.huangyuanlove/customView$id');
    final controller = CustomViewController._(
      _channel,
    );
    widget.onViewCreated(controller);
  }
}

```
这里的`OhosView`组件就是用来桥接PlatformView组件的。
其中：
> viewType：传递给Native侧，告知插件需要创建那个PlatformView，这个PlatformView需要在插件初始化时注册。
> onPlatformViewCreated：PlatformView创建成功时的回调。
> creationParams：传递给PlatformView的初始化参数。

这里需要注意，参数`viewType`必须和原生侧的`CustomPlugin`类中的`onAttachedToEngine`方法中，调用`registerViewFactory`方法第一个参数一致。
在`_onPlatformViewCreated`方法中注册的`Channel`就更不需要多说了.

### 展示并运行
我们找个页面来同时展示一下flutter组件和原生组件，这里为了简单，直接修改了`main.dart`

``` dart
import 'dart:math';

import 'package:flutter/material.dart';
import 'package:ohflutter_3221/widget/CustomOhosView.dart';

void main() {
  runApp(const MaterialApp(home: Main()));
}

class Main extends StatelessWidget {
  const Main({super.key});

  @override
  Widget build(BuildContext context) {
    return const Scaffold(
      body: CustomViewExample(),
    );
  }
}

class CustomViewExample extends StatefulWidget {
  const CustomViewExample({super.key});

  @override
  State<CustomViewExample> createState() => _CustomViewExampleState();
}

class _CustomViewExampleState extends State<CustomViewExample> {
  String receivedData = '';
  CustomViewController? _controller;

  void _onCustomOhosViewCreated(CustomViewController controller) {
    _controller = controller;
    _controller?.customDataStream.listen((data) {
      //接收到来自OHOS端的数据
      setState(() {
        receivedData = '来自ohos的数据：$data';
      });
    });
  }

  Widget _buildOhosView() {
    return Expanded(
      flex: 1,
      child: Container(
        color: Colors.blueAccent.withAlpha(60),
        child: CustomOhosView(_onCustomOhosViewCreated),
      ),
    );
  }

  Widget _buildFlutterView() {
    return Expanded(
      flex: 1,
      child: Column(
        mainAxisAlignment: MainAxisAlignment.center,
        mainAxisSize: MainAxisSize.max,
        children: [
          TextButton(
            onPressed: () {
              final randomNum = Random().nextInt(10);
              _controller?.sendMessageToOhosView('flutter - $randomNum ');
            },
            child: const Text('发送数据给ohos'),
          ),
          const SizedBox(height: 10),
          Text(receivedData),
        ],
      ),
    );
  }

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        _buildOhosView(),
        _buildFlutterView(),
      ],
    );
  }
}

```
这样我们就完成了原生组件的展示，和flutter组件的通信。

## 效果
放个效果图  
![platform_view](image/harmony_flutter/platform_view.gif)
