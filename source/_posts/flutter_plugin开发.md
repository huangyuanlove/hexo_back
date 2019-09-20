---
title: flutter_plugin开发
tags: [flutter]
date: 2019-09-16 11:20:05
keywords: [flutter_plugin,flutter插件开发,插件开发,flutter嵌入原生控件]
---

一个开发Flutter plugin  和 在Flutter中嵌入原生控件的笔记。完全是照着的官网来实践的。

https://flutter.dev/docs/development/packages-and-plugins/developing-packages

Flutter中的package分为两种，一种是纯dart语言的的package，比如 [`fluro`](https://pub.dev/packages/fluro)，称之为*Dart packages*

还有一种是由原生平台代码(Android:java or kotlin，iOS：OC or swift)，比如[`battery`](https://pub.dev/packages/battery)，称之为*Plugin packages*。

下面是对*Plugin packages*开发的实践

<!--more-->

### Create the package

创建工程，命令行

`flutter create --org com.example --template=plugin name`

默认Android是java，iOS是OC，也可以指定默认语言

`flutter create --template=plugin -i swift -a kotlin name`

也可以同过AS创建，选择`Flutter plugin`就好。

主要文件如下(以工程根目录为基础目录来说的)：

* lib

  刚创建好的项目中，该文件夹下只有一个以项目名命名的dart文件，创建了一个MethodChannel 和实现了一个`platformVersion`方法

* android

  这里需要注意一下，在AndroidStudio中以Project方式预览工程时，该目录下的src.main文件夹下只有一个清单文件，可以切换到Android方式预览。可以看到src.main下有一个实现`MethodCallHandler`接口的类，并且已经实现了`getPlatformVersion`的调用，我们开发插件原生代码就是在这里写的

* iOS

  在AndroidStudio下看到只有两个文件夹Assets和Classes，在Classes下有个`.h` 和`.m`，也是创建了`FlutterMethodChannel`和实现了`platformVersion`调用

* Example

  在这里测试我们自己写的插件，并且给使用这提供示例。该文件夹下包含Android、iOS和Flutter代码

### Implement the package

As a plugin package contains code for several platforms written in several programming languages, some specific steps are needed to ensure a smooth experience.

####  Define the package API

假如我们要开发一个打开某些原生界面的插件，比如打开原生设置页面、拨号页面、浏览器等

在`lib`下的dart文件中，增加一个名字为`openOSView`的方法调用，并且传递一个String类型的url参数

``` dart
class FlutterPluginOpenNative {
  static const MethodChannel _channel =
      const MethodChannel('flutter_plugin_open_native');

  static Future<String> get platformVersion async {
    final String version = await _channel.invokeMethod('getPlatformVersion');
    return version;
  }

  static Future<void>  openNative(String url) async {
    await _channel.invokeMethod('openOSView',url);
  }
}
```

#### Add Android platform code 

在Android下实现了`MethodCallHandler`接口的类中

``` java
@Override
  public void onMethodCall(MethodCall call, Result result) {
    if (call.method.equals("getPlatformVersion")) {
      result.success("Android " + android.os.Build.VERSION.RELEASE);
    } else if(call.method.equals("openOSView")){
      Intent intent = new Intent();
      intent.setAction(Intent.ACTION_VIEW);
      intent.setData(Uri.parse(call.arguments.toString()));
      activity.startActivity(intent);
    }
    else {
      result.notImplemented();
    }
  }
```

#### Add iOS platform code 

在iOS

``` objective-c
- (void)handleMethodCall:(FlutterMethodCall*)call result:(FlutterResult)result {
    if ([@"getPlatformVersion" isEqualToString:call.method]) {
        result([@"iOS " stringByAppendingString:[[UIDevice currentDevice] systemVersion]]);
    }
    else if([@"openOSView" isEqualToString:call.method]){
        NSString * phoneUri = call.arguments;
        [[UIApplication sharedApplication] openURL:[NSURL URLWithString:phoneUri]];
    }
    
    else {
        result(FlutterMethodNotImplemented);
    }
}
```

如果是用XCode开发的话，选择打开example-->ios-->Runner.xcworkspace，该文件存在于

`Pods-->Development Pods-->projectname`下

![project in xcode](/image/flutter/flutter_plugin_project_in_xcode.png)

第一个红框是项目中example-->ios中的内容

第二个红框是项目中ios文件夹下内容

-----

这样，一个简单的插件就开发完了，我们可以在example中测试一下，由于插件特别简单，不需要在原生侧做初始化之类的工作，使用创建项目时生成的代码就可以满足我们的需要。

在`example-->lib-->main.dart`中放一个按钮，点击的时候调用`FlutterPluginOpenNative.openNative("https://blog.huangyuanlove.com");`然后运行到设备，查看一下效果

`flutter run -d all` 可以运行到所有已连接的设备

----



### Flutter嵌入原生控件

上面是开发的插件，下面是如果在flutter中嵌入原生控件

这里用到了Flutter中的两个类`AndroidView`和`UiKitView`，懒得翻译直接看官方版https://api.flutter.dev/flutter/widgets/AndroidView-class.html  和https://api.flutter.dev/flutter/widgets/UiKitView-class.html

例如，我们要嵌入原生展示文字的控件，对于Android来讲，一般是TextView，对于iOS来讲，一般是UILabel。

#### 在 flutter侧

一般是一个Controller和一个StatefulWidget，当然这里的Controller只是一个普通类，用来创建channel和原生通信

代码如下

``` dart

typedef void TextViewCreatedCallback(TextViewController controller);
class TextViewController {
  TextViewController._(int id)
      : _channel = new MethodChannel('me.chunyu.textview/textview');

  final MethodChannel _channel;

  //调用setText方法给控件设置文字，需要原生侧进行实现
  Future<void> setText(String text) async {
    assert(text != null);
    return _channel.invokeMethod('setText', text);
  }
}

class AndroidTextView extends StatefulWidget {

  final TextViewCreatedCallback onTextViewCreated;
  const AndroidTextView({
    Key key,
    this.onTextViewCreated,
  }) : super(key: key);

  @override
  _AndroidTextViewState createState() => _AndroidTextViewState();
}

//对Android平台返回AndroidView，对iOS平台返回UiKitView，并且制定viewType
class _AndroidTextViewState extends State<AndroidTextView> {
  @override
  Widget build(BuildContext context) {
    if (defaultTargetPlatform == TargetPlatform.android) {
      return AndroidView(
        viewType: 'me.chunyu.textview/textview',
        onPlatformViewCreated: _onPlatformViewCreated,
      );
    }else if (defaultTargetPlatform == TargetPlatform.iOS){
      return UiKitView(
        viewType: 'me.chunyu.textview/textview',
        onPlatformViewCreated: _onPlatformViewCreated,
      );
    }
    return Text(
        '$defaultTargetPlatform is not yet supported by the text_view plugin');

  }
  void _onPlatformViewCreated(int id) {
    if (widget.onTextViewCreated == null) {
      return;
    }
    print("id _onPlatformViewCreated :--> $id");


    widget.onTextViewCreated(new TextViewController._(id));
  }
}
```

#### 在Android侧

我们需要一个实现了`PlatformViewFactory`的类 和一个实现了`PlatformView`、` MethodCallHandler `接口的类

``` java

class TextViewFactory extends PlatformViewFactory{

  private final BinaryMessenger messenger;

  public TextViewFactory(BinaryMessenger messenger) {
    super(StandardMessageCodec.INSTANCE);
    this.messenger = messenger;
  }

  @Override
  public PlatformView create(Context context, int id, Object o) {
    return new FlutterTextView(context, messenger, id);
  }

}

class FlutterTextView implements PlatformView, MethodCallHandler  {
  private final TextView textView;
  private final MethodChannel methodChannel;

  FlutterTextView(Context context, BinaryMessenger messenger, int id) {
    Log.e("id","id :-->" +id);
    textView = new TextView(context);
    textView.setTextColor(Color.BLUE);
    textView.setBackgroundColor(Color.GREEN);
    methodChannel = new MethodChannel(messenger, "me.chunyu.textview/textview");
    methodChannel.setMethodCallHandler(this);
  }

  @Override
  public View getView() {
    return textView;
  }

  //实现以下flutter通过channel调用的`setText`方法
  @Override
  public void onMethodCall(MethodCall methodCall, MethodChannel.Result result) {
    switch (methodCall.method) {
      case "setText":
        setText(methodCall, result);
        break;
      default:
        result.notImplemented();
    }

  }

  //设置文字，并返回成功
  private void setText(MethodCall methodCall, Result result) {
    String text = (String) methodCall.arguments;
    textView.setText(text);
    result.success(null);
  }

  @Override
  public void dispose() {}
}
```

这里需要注意，在插件的`registerWith`方法中注册一下`channel`

``` java
registrar.platformViewRegistry().registerViewFactory("me.chunyu.textview/textview",new TextViewFactory(registrar.messenger()));
```



#### 在iOS侧

一个`.h`文件，声明一下实现`NSObject<FlutterPlatformView>`

``` objc
#import <Foundation/Foundation.h>
#import <Flutter/Flutter.h>
NS_ASSUME_NONNULL_BEGIN

@interface AndroidTextView : NSObject<FlutterPlatformView>
- (instancetype)initWithWithFrame:(CGRect)frame
                   viewIdentifier:(int64_t)viewId
                        arguments:(id _Nullable)args
                  binaryMessenger:(NSObject<FlutterBinaryMessenger>*)messenger;

@end
@interface FlutterAndroidTextViewFactory : NSObject<FlutterPlatformViewFactory>

- (instancetype)initWithMessenger:(NSObject<FlutterBinaryMessenger>*)messager;

@end

NS_ASSUME_NONNULL_END
```

一个`.m`文件，实现以上方法

``` objc

#import "AndroidTextView.h"

@implementation FlutterAndroidTextViewFactory{
    NSObject<FlutterBinaryMessenger>*_messenger;
}

- (instancetype)initWithMessenger:(NSObject<FlutterBinaryMessenger> *)messager{
    self = [super init];
    if (self) {
        _messenger = messager;
    }
    return self;
}

-(NSObject<FlutterMessageCodec> *)createArgsCodec{
    return [FlutterStandardMessageCodec sharedInstance];
}

-(NSObject<FlutterPlatformView> *)createWithFrame:(CGRect)frame viewIdentifier:(int64_t)viewId arguments:(id)args{
    AndroidTextView * androidTextView = [[AndroidTextView alloc] initWithWithFrame:frame viewIdentifier:viewId arguments:args binaryMessenger:_messenger];
    
    return androidTextView;
    
}

@end


@implementation AndroidTextView{
    int64_t _viewId;
    FlutterMethodChannel* _channel;
    UILabel * _label;
    
}

- (instancetype)initWithWithFrame:(CGRect)frame viewIdentifier:(int64_t)viewId arguments:(id)args binaryMessenger:(NSObject<FlutterBinaryMessenger> *)messenger{
    if ([super init]) {
        _label = [[UILabel alloc] init];
        _label.textColor = UIColor.redColor;
        _label.backgroundColor = UIColor.blueColor;
        _label.font = [UIFont fontWithName:@"Arial" size:30];
        _viewId = viewId;
        NSString* channelName =  @"me.chunyu.textview/textview";
        _channel = [FlutterMethodChannel methodChannelWithName:channelName binaryMessenger:messenger];
        __weak __typeof__(self) weakSelf = self;
        [_channel setMethodCallHandler:^(FlutterMethodCall *  call, FlutterResult  result) {
            [weakSelf onMethodCall:call result:result];
        }];   
    }
    
    return self;
}

-(UIView *)view{
    NSLog(@"invoke view()");
    return _label;
}
//实现flutter侧通过channel调用的`setText`方法
-(void)onMethodCall:(FlutterMethodCall*)call result:(FlutterResult)result{
    if ([[call method] isEqualToString:@"setText"]) {
        _label.text = [call arguments];
        NSLog(@"%@", call.arguments);
    }
}
@end
```

在plugin中的`registerWithRegistrar`方法中注册一下`channel`

``` objc
[registrar registerViewFactory:[[FlutterAndroidTextViewFactory alloc] initWithMessenger:registrar.messenger] withId:@"me.chunyu.textview/textview"];
```

需要注意的是，如果想要在iOS中显示这种flutter嵌入原生的空间，需要在info.plist文件中加入

``` objc
<key>io.flutter.embedded_views_preview</key>
<true/>
```

在flutter中嵌入原生控件并不推荐，性能跟不上，消耗太大了，但是在某些情况下不得不这么搞。。。。。



### Adding documentation

这个没啥好说的，跟着官方做就好了

When you publish a package, API documentation is automatically generated and published to dartdocs.org, see for example the [device_info docs](https://pub.dev/documentation/device_info/latest)

### Adding licenses to the LICENSE file

添加协议

### Publishing packages

发布之前运行一下 `flutter pub pub publish --dry-run`，根据提示修改不满足需求的地方。

之后执行`flutter pub pub publish`，具体可以看For details on publishing, see the [publishing docs](https://dart.dev/tools/pub/publishing) for the Pub site.

