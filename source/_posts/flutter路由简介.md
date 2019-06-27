---
title: flutter路由简介
tags: [Flutter]
date: 2019-06-27 20:13:13
keywords: [Flutter路由,命名路由,Flutter界面传值]
---

管理多个页面时有两个核心概念和类：[Route](https://docs.flutter.io/flutter/widgets/Route-class.html)和 [Navigator](https://docs.flutter.io/flutter/widgets/Navigator-class.html)。 一个route是一个屏幕或页面的抽象，Navigator是管理route的Widget。Navigator可以通过route入栈和出栈来实现页面之间的跳转。所谓路由管理，就是管理页面之间如何跳转，通常也可被称为导航管理。这和原生开发类似，无论是Android还是iOS，导航管理都会维护一个路由栈，路由入栈(push)操作对应打开一个新页面，路由出栈(pop)操作对应页面关闭操作，而路由管理主要是指如何来管理路由栈。

<!--more-->

#### 示例

比如我们想要从A界面打开B界面，并且从B界面返回时需要携带数据给A界面，我们可以这么写

``` dart
Navigator.push(context, MaterialPageRoute(builder: (context) {
return SaveImageWidget();
})).then((onValue) {
print(onValue);
});
```

当我们从B界面返回的时候

``` dart
Navigator.of(context).pop("B界面返回的数据");
```

`MaterialPageRoute`继承自`PageRoute`类，`PageRoute`类是一个抽象类，表示占有整个屏幕空间的一个模态路由页面，它还定义了路由构建及切换时过渡动画的相关接口及属性。

#### Navigator

`Navigator`是一个路由管理的widget，它通过一个栈来管理一个路由widget集合。通常当前屏幕显示的页面就是栈顶的路由。方法大致分为两种：push:打开新页面，包括`pushAndRemoveUntil`,`pushNamed`,`pushNamedAndRemoveUntil`,`pushReplacement`,`pushReplacementNamed`,；pop：弹出当前界面,包括`popAndPushNamed`,`popUntil`

##### 命名路由

这里需要说一下命名路由，上面的打开新界面的时候，push方法中参数是MaterialPageRoute，还有一种是给定路由字符串，在此之前，我们需要先注册路由表：

找到flutter工程的入口main方法，找到`MaterialApp`,添加`routers`属性，

![注册路由表](/image/flutter/register_routers.png)

代码在这里 <https://github.com/huangyuanlove/test_flutter/blob/master/lib/main.dart>

具体页面代码在这里 <https://github.com/huangyuanlove/test_flutter/tree/master/lib/test_named_router>

#### 使用

```dart
Navigator.of(context).pushNamed('a_router_widget');
```

> 这种方法只是简单的将我们需要进入的页面push到栈顶，以此来显示当前页面,其参数是一个字符串类型，传入的是页面对应的路由名称

```dart
Navigator.of(context).pushReplacementNamed('d_router_widget');
```

> 这种方法是将当前界面替换成要跳转的界面，比如A打开B，B使用这种方法打开C，当从C界面返回的时候，会直接返回A。

```dart
Navigator.of(context).popAndPushNamed('d_router_widget');
```

> 指将当前页面pop，然后跳转到指定页面（将当前路由弹出导航器，并将命名路由推到它的位置。）

```dart
Navigator.of(context).pushNamedAndRemoveUntil('d_router_widget', (Route<dynamic> route) => false);
```

> 指将指定的页面加入到路由中，然后将其他所有的页面全部pop, (Route<dynamic> route) =>  false将确保删除推送路线之前的所有路线。
> Push the route with the given name onto the navigator, and then remove all
> the previous routes until the `predicate` returns true.


```dart
Navigator.of(context).pushNamedAndRemoveUntil('d_router_widget', ModalRoute.withName('a_router_widget'));
```

> 
> Push the route with the given name onto the navigator, and then remove all
> the previous routes until the `predicate` returns true.
> 

```dart
Navigator.of(context).maybePop();
```

> maybePop 会自动进行判断，如果当前页面pop后，会显示其他页面，不会出现问题，则将执行当前页面的pop操作，否则将不执行。

```dart
Navigator.of(context).canPop();
```

> 
> 判断当前页面能否进行pop操作，并返回bool值
> 

```dart
Navigator.of(context).pop();
```

> 直接退出当前页面

```dart
Navigator.popUntil(context, ModalRoute.withName('a_router_widget'));
```

> 弹出界面，直到遇到第一个名字为给name的路由界面

#### 命名路由传参

现在命名路由也支持 参数传递了

``` dart
FlatButton(onPressed: () {
              Map arguments = {
              "shareContent": "这是分享的内容",
              "shareImageUrl": "这是分享的图片链接",
              "shareTitle": "这是分享的标题"
              };
              Navigator.of(context).pushNamed("c_router_widget",arguments: arguments).then((value){
                print("这是B界面打开 c_router_widget 接收到的Value$value");
              });
            }, child: Text("命名路由C"))
```

我们在注册路由的时候可以这么写

``` dart
"c_router_widget": (BuildContext context) {
          Map arguments = ModalRoute.of(context).settings.arguments;
          return CRouterWidget(
              shareContent: arguments["shareContent"],
              shareImageUrl: arguments["shareContent"],
              shareTitle: arguments["shareTitle"]);
        },
```

也就是使用`ModalRoute.of(context).settings.arguments`来获取参数，或者在具体界面中获取也一样。



----

以上