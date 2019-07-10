---
title: Flutter InheritedWidget和Notification
tags: [Flutter]
date: 2019-07-08 14:14:06
keywords: [InheritedWidget,状态共享]
---

InheritedWidget是Flutter中非常重要的一个功能型Widget，它可以高效的将数据在Widget树中向下传递(只能向下传递，无法向上传递，如果需要向上传递可以使用Notification)、共享，这在一些需要在Widget树中共享数据的场景中非常方便，如Flutter中，正是通过InheritedWidget来共享应用主题(Theme)和Locale(当前语言环境)信息的。这里建议阅读以下`theme.dart`的源码

<!--more-->

现在有一需求，从A界面打开B界面，在B界面对数据进行修改，返回A界面的时候，A界面显示的数据也是修改之后的。

两种方式：第一种我们可以在B界面修改数据之后做持久化存储，或者修改的数据做成全局静态字段，返回A界面后由A界面刷新

第二种就是使用InheritedWidget进行数据共享实现。

### InheritedWidget

#### 创建数据model和共享Widget

``` dart
class InheritedTestModel {
  String name;
  int age;
  String sex;
}
```

#### 继承InheritedWidget来共享数据

这里我们需要继承`InheritedWidget`，并且声明一个静态方法，用来从BuildContext中获取共享数据。

还需要重写updateShouldNotify方法，如果返回true，则子树中依赖(build函数中有调用)本widget的子widget的`state.didChangeDependencies`会被调用。


下面是详细代码：

``` dart
class InheritedModelContext extends InheritedWidget {
  final InheritedTestModel model;

  InheritedModelContext({Key key, @required this.model, @required Widget child})
      : super(key: key, child: child);

  static InheritedModelContext of(BuildContext context) {
    return context.inheritFromWidgetOfExactType(InheritedModelContext);
  }

  @override
  bool updateShouldNotify(InheritedModelContext oldWidget) {
    return model.age != oldWidget.model.age ||
        model.name != oldWidget.model.name ||
        model.sex != oldWidget.model.sex;
  }
}
```

一个约定俗成的 `of`方法用来让子树获取到共享数据。至于为什么可以获取的数据，

> 在 `Element` 的内部有一个 `Map<Type, InheritedElement> _inheritedWidgets;` 参数，**_inheritedWidgets 一般情况下是空的，只有当父控件是 InheritedWidget 或者本身是 InheritedWidgets 时，它才会有被初始化，而当父控件是 InheritedWidget  时，这个 Map 会被一级一级往下传递与合并。**
>
> 所以当我们通过 `context` 调用 `inheritFromWidgetOfExactType` 时，就可以通过这个 `Map`  往上查找，从而找到这个上级的 `InheritedWidget` 。
>
>
> 作者：恋猫de小郭链接：https://juejin.im/post/5d0634c7f265da1b91639232来源：掘金著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。

#### 在A界面创建数据，并共享到下一个界面

我们首先在界面A创建一个共享的数据model，显示在界面上，然后通过上面定义的`InheritedModelContext`共享给下一个界面：

``` dart
class InheritedWidgetTestRouteState extends State<InheritedWidgetTestRoute> {
  InheritedTestModel model;

  @override
  void initState() {
    model = new InheritedTestModel();
    model.age = 10;
    model.name = "abc";
    model.sex = "f";
    super.initState();
  }
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text("InheritedWidget"),
      ),
      body: Center(
        child: Column(
          children: <Widget>[
            Text("age: ${model.age}"),
            Text("name: ${model.name}"),
            Text("sex: ${model.sex}"),
          ],
        ),
      ),
      floatingActionButton: FloatingActionButton(
        onPressed: () {
          Navigator.of(context).push(MaterialPageRoute(builder: (context){
          return  InheritedModelContext(
            model: model,
            child: InheritedSecondPage(),
          );
          }));
        },
        child: Icon(Icons.navigate_next),
      ),
    );
  }
}
```

#### 在B界面获取共享数据，并作出修改

我们在B界面获取到共享数据并展示出来，通过点击按钮，改变共享数据的值，当我们返回上一个界面的时候会发现上个界面的显示也被改变了

``` dart

class InheritedSecondPageState extends State<InheritedSecondPage> {
  InheritedModelContext modelContext;
  InheritedTestModel model;

  String name = "";
  int age = 0;
  String sex = "";

  @override
  void initState() {
    super.initState();
  }

  @override
  Widget build(BuildContext context) {
    modelContext = InheritedModelContext.of(context);
    model = modelContext.model;
    return Scaffold(
      appBar: AppBar(
        title: Text("第二页"),
      ),
      body: Center(
        child: Column(
          children: <Widget>[
            Text("age: ${model.age}"),
            Text("name: ${model.name}"),
            Text("sex: ${model.sex}"),
            Column(
              children: <Widget>[
                FlatButton(
                  onPressed: () {
                    setState(() {
                      model.name = DateTime.now().toLocal().toString();
                    });
                  },
                  child: Text("改变Name"),
                ),
                FlatButton(
                  onPressed: () {
                    setState(() {
                      model.age = DateTime.now().millisecondsSinceEpoch;
                    });
                  },
                  child: Text("改变Age"),
                ),
                FlatButton(
                  onPressed: () {
                    setState(() {
                      model.sex = DateTime.now().toString();
                    });
                  },
                  child: Text("改变sex"),
                ),
              ],
            ),
          ],
        ),
      ),
    );
  }
}
```



### Notification

> Notification是Flutter中一个重要的机制，在Widget树中，每一个节点都可以分发通知，通知会沿着当前节点（context）向上传递，所有父节点都可以通过NotificationListener来监听通知，Flutter中称这种通知由子向父的传递为“通知冒泡”（Notification Bubbling），这个和用户触摸事件冒泡是相似的，但有一点不同：通知冒泡可以中止，但用户触摸事件不行。
>
> 源自  ：https://book.flutterchina.club/chapter8/notification.html

#### 创建Model、InheritedWidget和上面一致

#### 定义Notification

```dart
class MyNotification extends Notification{
  final String msg;
  MyNotification(this.msg);
}
```

#### 定义两个Widget，放在同一个界面中

```dart
class WidgetA extends StatefulWidget {
  @override
  State<StatefulWidget> createState() {
    return WidgetAState();
  }
}

class WidgetAState extends State<WidgetA> {

  @override
  void initState() {
    super.initState();

  }

  @override
  Widget build(BuildContext context) {
    final InheritedContext inheritedContext = InheritedContext.of(context);
    final SinglePageModel model = inheritedContext.model;
    print("WidgetA build");
    return Center(
      child: Column(
        children: <Widget>[
          Text("当前页${model.page}"),
          Text("内容：${model.content}"),
          FlatButton(
            onPressed: () {
              if(model.page==null){
                model.page = 1;
              }else {
                model.page += 1;
              }
              MyNotification("refresh").dispatch(context);
              setState(() {

              });


            },
            child: Icon(Icons.add),
          ),
          FlatButton(
            onPressed: () {
              model.content = DateTime.now().toLocal().toString();
              //发送通知
              MyNotification("refresh").dispatch(context);
              setState(() {

              });
            },
            child: Text("改变content"),
          ),
        ],
      ),
    );
  }
}
```

#### 创建界面并且监听自定义的通知

```dart
class InheritedWidgetInSinglePage extends StatefulWidget {
  @override
  State<StatefulWidget> createState() {
    return InheritedWidgetInSinglePageState();
  }
}

class InheritedWidgetInSinglePageState
    extends State<InheritedWidgetInSinglePage> {
  SinglePageModel model = SinglePageModel();

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text("同一页面共享数据"),
      ),
      body: NotificationListener<MyNotification>(
      //监听通知
        onNotification: (notification) {
          setState(() {});
        },
        child: InheritedContext(
          model: model,
          child: Column(
            children: <Widget>[
              Text("page:${model.page}"),
              Text("content:${model.content}"),
              FlatButton(
                onPressed: () {
                  if (model.page == null) {
                    model.page = 1;
                  } else {
                    model.page += 1;
                  }
                  setState(() {});
                },
                child: Icon(Icons.add),
              ),
              WidgetA(),
              WidgetB(),
            ],
          ),
        ),
      ),
      floatingActionButton: FloatingActionButton(
        onPressed: () {
          Navigator.of(context).push(MaterialPageRoute(builder: (context) {
            return InheritedContext(
              model: model,
              child: SecondPageWidget(),
            );
          }));
        },
        child: Icon(Icons.navigate_next),
      ),
    );
  }
}
```

这里的`SecondPageWidget`和上面提到的B界面差不多，都是获取到共享数据后展示，点击按钮做出改变。

-----

以上