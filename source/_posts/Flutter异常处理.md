---
title: Flutter异常处理
tags: [Android,Flutter]
date: 2020-07-27 21:04:53
keywords: [flutter异常,flutter异常捕获,flutter异常统一处理]
---

Flutter异常和Java异常类似，都是代码运行时发生的错误事件，我们可以通过与Java类似的try-catch机制来捕获这个异常，和java不同的地方在于 Dart 采用事件循环的机制来运行任务，各个任务的运行状态是互相独立的，也就是说，即便某个任务出现了异常我们没有捕获它，Dart 程序也不会退出，只会导致当前任务后续的代码不会被执行，用户仍可以继续使用其他功能。

<!--more-->

dart中的异常分为App异常和Framework异常，根据来源不同，捕获方式也不同。App异常我们可以通过try--catch或者异步调用中的catchError捕获；Framework中的异常可以通过自定义ErrorWidget.builder进行捕获+展示

### App异常的捕获方式

App异常，就是应用代码的异常，通常由未处理应用层其他模块所抛出的异常引起。根据异常代码的执行时序，App异常可以分为两类，即同步异常和异步异常:同步异常可以通过try-catch机制捕获，异步异常则需要采用Future 提供的catchError语句捕获。

``` dart
  //异常捕获
  try{
    throw StateError("this is a dart exception");

  }catch (e){
    print(e);
  }


  Future.delayed(Duration(seconds: 1))
  .then((value) => throw StateError("this is a dart exception in future") )
  .catchError((e)=>print(e));


  //无法捕获
  try{
    Future.delayed(Duration(seconds: 1))
        .then((value) => throw StateError("this is a dart exception in future by try catch") );
  }catch(e){
    print(e);
  }
```

需要注意的是，这两种方式是不能混用的。无法使用try- -catch去捕获一个异步调用所抛出的异常的。如果我们想集中管理代码中的所有异常，Flutter 也提供了Zone.runZoned 方法。我们可以给代码执行对象指定一-个Zone，在Dart中，Zone表示一个代码执行的环境范围，其概念类似沙盒，不同沙盒之间是互相隔离的。如果我们想要观察沙盒中代码执行出现的异常，沙盒提供了onError回调函数，拦截那些在代码执行对象中的未捕获异常。
在下面的代码中，我们将可能抛出异常的语句放置在了Zone里。可以看到，在没有使用try-catch和catchError的情况下，无论是同步异常还是异步异常，都可以通过Zone直接捕获到:

``` dart
  //runZoned
  runZoned(() {
    throw StateError("this is a dart exception in future");
    
  }, onError: (dynamic e, StackTrace stack) {
    print(e);
    print("catch exception by zone");
  });


  runZoned(() {
    Future.delayed(Duration(seconds: 1))
        .then((value) =>
    throw StateError("this is a dart exception in future by try catch"));
  }, onError: (dynamic e, StackTrace stack) {
    print(e);
    print("catch future exception by zone");
  });
```



## Framework 异常捕获

Framework异常，就是Flutter框架引发的异常，通常是由应用代码触发了Flutter 框架底层的异常判断引起的。比如，当布局错误时(具体一点就是Text控件的构造方法传入null)，Flutter 就会自动展示一个包含错误信息的红色页面，这其实是因为，Flutter 框架在调用build方法构建页面时进行了try- -catch的处理，并提供了一个ErrorWidget，用于在出现异常时进行信息提示:

``` dart
//abstract class ComponentElement extends Element 
//StatelessElement extends 	 ComponentElement
// StatefulElement extends 	 ComponentElement

built = ErrorWidget.builder(
        _debugReportException(
          ErrorDescription('building $this'),
          e,
          stack,
          informationCollector: () sync* {
            yield DiagnosticsDebugCreator(DebugCreator(this));
          },
        )
```

这个页面反馈的信息比较丰富，适合开发期定位问题。但我们并不想让用户看到这种错误信息，希望给用户展示一个更加友好的页面。因此，我们通常会重写ErrorWidget.builder方法，返回我们自定义的展示信息，下面我们直接返回了一个居中的Text控件:

``` dart
 ErrorWidget.builder = (FlutterErrorDetails flutterErrorDetails) {
      return Scaffold(
        body: Center(
          child: Text("${flutterErrorDetails.exception}"),
        ),
      );
    };
```



#### 异常统一处理

为了集中处理框架异常，Flutter 提供了FlutterError类，这个类的onError属性会在接收到框架异常时执行相应的回调。因此，要实现自定义捕获逻辑，我们只要为它提供一个自定义的错误处理回调即可。
在下面的代码中，我们使用Zone提供的handleUncaughtError语句，将Flutter框架的异常统- -转发到当前的Zone中，这样我们就可以统一使用Zone去处理应用内的所有异常了:

``` dart

void main() {
  FlutterError.onError = (FlutterErrorDetails details) async {
    //转发至zone
    Zone.current.handleUncaughtError(details.exception, details.stack);
  };

  runZoned<Future<Null>>(() async {
    runApp(MyApp());
  }, onError: (error, stackTrace) async {
    //在这里处理异常 搞个channel，转发到原生，上报到bugly平台
    print("-------");
    print(error);
    print(stackTrace);
    print("-------");
  });
}
```



----

以上