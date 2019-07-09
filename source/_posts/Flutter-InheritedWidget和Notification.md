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

#### 创建数据model和共享Widget

``` dart
class InheritedTestModel {
  String name;
  int age;
  String sex;
}
```

这里我们需要继承`InheritedWidget`，并且声明一个静态方法，用来从BuildContext中获取共享数据。

还需要重写updateShouldNotify方法，来判断什么时候进行更新

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

