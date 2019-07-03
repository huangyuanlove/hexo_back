---
title: Flutter中的provider
tags: [Android,Flutter]
date: 2019-07-01 11:28:45
keywords: [状态共享,BLoC和provider,provider]
---

作为一个状态共享的解决方案，**不复杂，好理解，代码量不大的情况下，可以方便组合和控制刷新颗粒度** ， 而原 Google 官方仓库的状态管理 [flutter-provide](https://link.juejin.im/?target=https%3A%2F%2Fgithub.com%2Fgoogle%2Fflutter-provide) 已宣告GG ， [provider](https://link.juejin.im/?target=https%3A%2F%2Fgithub.com%2FrrousselGit%2Fprovider) 成了它的替代品。和`scoped_moded`比起来，入侵性比较小，也适合比较复杂的应用场景。

具体的代码在 https://github.com/huangyuanlove/test_flutter/tree/master/lib/provider 

<!--more-->

#### 添加依赖

在pubspec.yaml中添加provider镜像：具体版本号可以在`https://pub.dev/packages/provider#-installing-tab-`查看

``` yaml
dependencies:
  flutter:
    sdk: flutter

  # The following adds the Cupertino Icons font to your application.
  # Use with the CupertinoIcons class for iOS style icons.
  cupertino_icons: ^0.1.2
  provider: ^3.0.0+1
dev_dependencies:
  flutter_test:
    sdk: flutter
```

#### 创建数据model

``` dart
import 'package:flutter/material.dart';

class CounterModel with ChangeNotifier{
  int _count = 0;

  int get value => _count;

  void increment(){
    _count ++;
    notifyListeners();
  }
}
```

这里使用了 mixin 混入了 `ChangeNotifier`，这个类能够帮驻我们自动管理所有听众。当调用`notifyListeners()` 时，它会通知所有听众进行刷新。

为了能更加直观的看到每个widget的build过程，做个包装的widget，不是依靠`debugPrintLayouts = true;`

``` dart
import 'package:flutter/material.dart';

class TestTextWidget extends StatelessWidget{

  TestTextWidget({@required this.logTag,@required this.child});
  final String logTag;
  final Widget child;


  @override
  Widget build(BuildContext context) {
    print(logTag);
    return child;
  }

}
```

在每次build的时候打印一下日志,并且在源码的`text.dart`中的build方法打印了一下内容`print("Text build:${data}");`

#### 如何传递、共享数据

##### 向子节点传递数据

我们在主页中加个按钮，点击后进入计数界面

``` dart
import 'package:provider/provider.dart';
FlatButton(
              child: Text("TestProvider"),
              onPressed: () {
                final counter = CounterModel();
                final textSize = 48;
                Navigator.push(context, MaterialPageRoute(builder: (context) {
                  return Provider<int>.value(
                    value: textSize,
                    child: ChangeNotifierProvider.value(
                      value: counter,
                      child: CounterFirstScreen(),
                    ),
                  );
                })).then((onValue) {
                  print(onValue);
                });
              },
            ),
```

这里我们向子节点传递了两个值：`textSize`和`counter`，对于需求来讲，`textSize`是固定不变的，可以通过`Provider<T>.value`来传入，而`counter`是我们需要多个界面共享的数据(某个界面改变其中的字段，其他界面也需要跟着改变)。使用`ChangeNotifierProvider<T>.value`来传入和共享，

- 1、 **Provider**  的内部 `DelegateWidget` 是一个 `StatefulWidget` ，所以可以更新且具有生命周期。
- 2、状态共享是使用了 `InheritedProvider` 这个 `InheritedWidget` 实现的。
- 3、巧妙利用 `MultiProvider` 和 `Consumer` 封装，实现了组合与刷新颗粒度控制。

具体可以看 https://juejin.im/post/5d0634c7f265da1b91639232

和 https://juejin.im/post/5d00a84fe51d455a2f22023f

#### 在子节点中获取数据

我们在`CounterFirstScreen`的build方法中获取上个界面传进来的值，并且在界面上以不同的方式显示出来

``` dart
class CounterFirstScreen extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    final _counter = Provider.of<CounterModel>(context);
    final _textSize = Provider.of<int>((context)).toDouble();
    print('first screen rebuild');
    return Scaffold(
      appBar: AppBar(
        title: Text('FirstPage'),
      ),
      body: Center(
        child: Column(
          mainAxisSize: MainAxisSize.max,
          children: <Widget>[
            TestTextWidget(
              logTag: "first page text counter",

              child: Text(
                'Value: ${_counter.value}',
                style: TextStyle(fontSize: _textSize),
              ),
            ),
            TestTextWidget(
              logTag: "first page text fix",

              child: Text(
                "固定文本，不需要重绘"
              ),
            ),
          ],
        ),
      ),
      floatingActionButton: FloatingActionButton(
        onPressed: () => {
              Navigator.of(context).push(MaterialPageRoute(builder: (context) {
//                return Provider<int>.value(
//                  value: _textSize.toInt(),
//                  child: ChangeNotifierProvider.value(
//                    value: _counter,
//                    child: Provider<Color>.value(
//                      value: Colors.red.shade50,
//                      child: SecondPage(),
//                    ),
//                  ),
//                );
              return MultiProvider(
                providers: [
                  Provider<int>.value(value: _textSize.toInt()),
                  ChangeNotifierProvider.value(value: _counter),
                  Provider.value(value: Colors.red.shade50),

                ],
                child: SecondPage(),
              );
              }))
            },

        child: Icon(Icons.navigate_next),
      ),
    );
  }
}
```

在build中我们通过`Provider.of<T>(context);`来获取顶层数据，这里泛型T指定了获取距离该界面最近的存储了T的祖先节点的数据。这里强烈建议在传入和获取值时加上泛型，经测试，传入相同类型的值，后面的回覆盖前面的。

当我们点击floatingActionButton的时候进入第二个界面，这里传入的数据比较多，可以通过MultiProvider或者嵌套Provider来做，在或者，把要传入的值封装成一个model来传入。在我们的业务上来看，并没有什么固定值需要以Provider的方式来传递，可以通过构造方法。

##### Consumer

我们在第二个界面中使用Consumer来获取共享数据，来达到控制局部刷新的目的。

我们需要在第二个界面中使用上一个界面中传过来的字体大小、文字内容、文字颜色。

``` dart
Consumer<int>(
              builder: (context, int textSize, _) {
                return TestTextWidget(
                  logTag: "secondPage.Consumer<int>",
                  child: Text(
                    "textSize ${textSize + 5}",
                    style: TextStyle(fontSize: (textSize + 5).toDouble()),
                  ),
                );
              },
            ),
```

``` dart
Consumer3<CounterModel, int, Color>(
              builder: (context, CounterModel counter, int textSize,
                  Color color, _) {
                return TestTextWidget(
                  logTag: "SecondPage Consumer3<CounterModel, int, Color>",
                  child: Text(
                    'Value:${counter.value}',
                    style:
                        TextStyle(fontSize: textSize.toDouble(), color: color),
                  ),
                );
              },
            ),
```

Consumer 使用了 [**Builder**](https://link.juejin.im/?target=https%3A%2F%2Fen.wikipedia.org%2Fwiki%2FBuilder_pattern) 模式，收到更新通知就会通过 builder 重新构建。`Consumer<T>` 代表了它要获取祖先中的哪种Model。

从源码中可以看出，作者帮我们做到了 Consumer6。。。。。。并且还能看出来，Consumer中就是使用Provider实现的，它的经典之处在于能够在复杂项目中，**极大地缩小你的控件刷新范围**。`Provider.of<T>(context)` 将会把调用了该方法的 context 作为听众，并在 `notifyListeners` 的时候通知其刷新。

我们上面也提到了在Text的build方法中加了日志，当我们进入第一页的时候，控制台输出：

``` verilog
I/flutter (26320): first screen rebuild                                 
I/flutter (26320): tag:第一个界面中显示计数的TestTextWidget             
I/flutter (26320): Text build:Value: 0                                  
I/flutter (26320): tag:第一个界面中固定的文本                           
I/flutter (26320): Text build:固定文本，不需要重绘                      
I/flutter (26320): Text build:FirstPage  
```

进入第二个界面的时候：

``` verilog
I/flutter (26320): second screen rebuild
I/flutter (26320): SecondPage Consumer3<CounterModel, int, Color>
I/flutter (26320): Text build:Value:0
I/flutter (26320): secondPage.Consumer<int>
I/flutter (26320): Text build:textSize 53
I/flutter (26320): tag:第二个界面中固定不变的TestTextWidget
I/flutter (26320): Text build:第二个界面的固定文本 TestTextWidget.child
I/flutter (26320): Text build:SecondPage fix text
I/flutter (26320): Text build:Second Page
I/flutter (26320): first screen rebuild
I/flutter (26320): tag:第一个界面中显示计数的TestTextWidget
I/flutter (26320): Text build:Value: 0
I/flutter (26320): tag:第一个界面中固定的文本
I/flutter (26320): Text build:固定文本，不需要重绘
I/flutter (26320): Text build:FirstPage
```

在第二个界面点击加号的时候

``` verilog

I/flutter (26320): first screen rebuild
I/flutter (26320): tag:第一个界面中显示计数的TestTextWidget
I/flutter (26320): Text build:Value: 1
I/flutter (26320): tag:第一个界面中固定的文本
I/flutter (26320): Text build:固定文本，不需要重绘
I/flutter (26320): Text build:FirstPage
I/flutter (26320): SecondPage Consumer3<CounterModel, int, Color>
I/flutter (26320): Text build:Value:1
```

再次点击加号
``` verilog
I/flutter (26320): first screen rebuild
I/flutter (26320): tag:第一个界面中显示计数的TestTextWidget
I/flutter (26320): Text build:Value: 2
I/flutter (26320): tag:第一个界面中固定的文本
I/flutter (26320): Text build:固定文本，不需要重绘
I/flutter (26320): Text build:FirstPage
I/flutter (26320): SecondPage Consumer3<CounterModel, int, Color>
I/flutter (26320): Text build:Value:2
```

可以看出来，第一个界面是全部重建可一次，而在第二个界面中，只有监听了CountModel的Text重建了。

---------

以上