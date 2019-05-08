---
title: Flutter BLoC 简单使用
tags: [Android,flutter]
date: 2019-05-08 14:34:49
keywords: 跨平台,Android,flutter
---

Flutter的设计灵感部分来自于React，主要是数据与视图分离，由数据来驱动视图的渲染。而对于我们在实际工程中的应用，就目前状态来讲，只是用来做UI，并没有用Flutter来做多少业务逻辑，涉及到的逻辑也不过是界面之间的数据、状态传递等。但并不排除将来会将重心稍微往Flutter侧偏移。

目前使用StatefulWidget完全可以适应目前的需求。但是需要考虑到后续扩展，需要找一种能够解决状态同步问题的方案。在了解了几种方案后确定使用BLoC。

https://juejin.im/post/5bac54c45188255c681589d3

https://www.jianshu.com/p/e7e1bced6890

https://www.jianshu.com/p/7573dee97dbb

<!--more-->

#### Stream

Stream看起来和rx家族东西东西很像，我们可以通过**StreamController**的**sink**传输一些数据，然后监听**StreamSubscription**流来感知数据，甚至可以通过**StreamTransformer**对数据流进行操作。当然也可以通过Flutter提供的**StreamBuilder**来构建Widget，

``` dart
Center(
        child: StreamBuilder<int>(
            stream: bloc.stream,
            initialData: bloc.value,
            builder: (BuildContext context, AsyncSnapshot<int> snapshot) {
              return Text(
                'You hit me: ${snapshot.data} times',
                style: Theme.of(context).textTheme.display1,
              );
            }),
      )
```

这里的snapshot则是通过sink传输过来的数据，然后显示在Text控件中。

一个简单的Stream演示

``` dart
import 'dart:async';

void main() {
  StreamController streamController = StreamController();

  StreamSubscription subscription =
      streamController.stream.listen((event){
        print("$event");
      });
  
  subscription.onData((data) {
    print("-->$data");
  });
  subscription.onDone(() => {print("on done")});

  streamController.sink.add("abc");
  streamController.sink.add("123");
  streamController.sink.add("def");

  streamController.close();
}
```

我们会得到这样一个输出：

``` xml
-->abc
-->123
-->def
on done
```

当然如果我们没有再次重写`subscription.onData`方法，则会执行`print("$event");`方法。

**Stream**有两种类型：**单订阅Stream和广播Stream**。单订阅Stream只允许在该Stream的整个生命周期内使用单个监听器，即使第一个subscription被取消了，你也没法在这个流上监听到第二次事件；而广播Stream允许任意个数的subscription，你可以随时随地给它添加subscription，只要新的监听开始工作流，它就能收到新的事件。

#### 简化版的demo

官方的计数器demo的简化版。

``` dart
class CountBLoC  {
  int _count = 0;
  final _controller = StreamController<int>();
  Stream<int> get stream => _controller.stream;
  int get value => _count;
  increment() {
    _controller.sink.add(++_count);
  }
}
```

``` dart
class CounterWidget extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    final bloc = CountBLoC();
    return Scaffold(
      appBar: AppBar(
        title: Text('Top Page'),
      ),
      body: Center(
        child: StreamBuilder<int>(
            stream: bloc.stream,
            initialData: bloc.value,
            builder: (BuildContext context, AsyncSnapshot<int> snapshot) {
              return Text(
                'You hit me: ${snapshot.data} times',
                style: Theme.of(context).textTheme.display1,
              );
            }),
      ),
      floatingActionButton: FloatingActionButton(
        onPressed: () => bloc.increment(),
        child: Icon(Icons.add),
      ),
    );
  }
}
```

1. 首先我们定义了一个BLoC类，在里面声明了一个**StreamController**和一个`increment()`方法，当我们调用increment()方法的时候，则会将计数加1，然后丢到sink里面，这样我们在任何监听`streamController.stream`的地方都将会收到这个数据，然后进行之后的工作，在上面的代码中，监听者是`StreamBuilder`,当有数据流入的时候，它会进行UI重绘，将数据显示到控件上。
2. 我们在点击按钮的时候，调用了`bloc.increment()`方法，产生了数据流。

这很像mvp模式中的P层的作用，业务逻辑处理都在这里面进行，然后通知UI重绘，当然这只是一个很粗糙的样例。

#### 关于Bloc的可访问性

以上的功能都是基于BLoC进行的，所以BLoC的可访问性需要得到保证

1. **全局单例（global Singleton）**：这种方式很简单，但是不推荐，因为Dart中对类没有析构函数（destructor）的概念，因此资源永远无法释放。

2. **局部变量（local instance）**：你可以创建一个Bloc局部实例，在某些情况下可以完美解决问题。但是美中不足的是，你需要在StatefulWidget中初始化，并记得在`dispose()`中释放它。

3. **由祖先（ancestor）来提供**：这也是最常见的一种方法，通过一个实现了StatefulWidget的父控件来获取访问权。

在大佬写的项目中有这种实现：

``` dart
// 所有 BLoCs 的通用接口
abstract class BlocBase {
  void dispose();
}

// 通用 BLoC provider
class BlocProvider<T extends BlocBase> extends StatefulWidget {
  BlocProvider({
    Key key,
    @required this.child,
    @required this.bloc,
  }): super(key: key);

  final T bloc;
  final Widget child;

  @override
  _BlocProviderState<T> createState() => _BlocProviderState<T>();

  static T of<T extends BlocBase>(BuildContext context){
    final type = _typeOf<BlocProvider<T>>();
    BlocProvider<T> provider = context.ancestorWidgetOfExactType(type);
    return provider.bloc;
  }

  static Type _typeOf<T>() => T;
}

class _BlocProviderState<T> extends State<BlocProvider<BlocBase>>{
  @override
  void dispose(){
    widget.bloc.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context){
    return widget.child;
  }
}
```

当我们使用的时候，

1. 自己定义处理逻辑的BLoC继承BLoCBase(这是为了能释放掉Stream资源)

2. 之前我们定义了一个界面叫Joke，使用的时候直接用`new Joke()`就好了，现在我们需要

3. ```dart
   BLoCProvider<JokeBLoC>(
     bloc: JokeBLoC(),
     child: Joke(),
   )
   ```

当然为了能够使用BLoC,需要对Joke进行改造，改造之后是`JokeWithBLoC`,于是就成了这样

``` dart
BLoCProvider<JokeBLoC>(
  bloc: JokeBLoC(),
  child: JokeWithBLoC(),
)
```

``` dart

class JokeBLoC extends BLoCBase{

  //处理数据返回
  StreamController<List<JokeBean>> _resultController = StreamController.broadcast();
  Stream<List<JokeBean>> get outResultList => _resultController.stream;
 
  //处理刷新和加载更多
  StreamController<bool> _indexController = StreamController<bool>.broadcast();
  Sink<bool> get inJokesIndex => _indexController.sink;

  //保存数据
  List<JokeBean> datas = [];
  var pageNumber = 1;

  //构造函数，开始监听分页信息，因为在JokeWithBLoC创建的时候就添加了数据
  NewsBLoC() {
    _indexController.stream.listen(_handleIndex);
    //错误处理
    _resultController.addError(
            (error) => print("_jokeResultController error->${error.toString()}"));
    _indexController.addError(
            (error) => print("_jokeIndexController error->${error.toString()}"));
  }

//加载数据
  void _handleIndex(bool isLoadMore) async {
    if (isLoadMore) {
      pageNumber++;
    } else {
      pageNumber = 1;
    }

    String dataUrl =
        "https://i.jandan.net/?oxwlxojflwblxbsapi=jandan.get_duan_comments&page=$pageNumber";
    try {
      Response<Map<String, dynamic>> response = await Dio().get(dataUrl);
      if (response.statusCode == 200) {
        var jokeModel = JokeModel.fromJson(response.data);
          if(isLoadMore){
            datas.addAll(jokeModel.comments);
          }else{
            datas = jokeModel.comments;
          }
        //数据返回后添加到流中，这样JokeWithBLoC中StreamBuild会被调用
          _resultController.add(datas);

      } else {
       showError();
      }
    } catch (e) {
      print(e.toString());
    }
  }
    @override
  void dispose() {
    //在这里关闭流
  }

}
```

``` dart

class JokeWithBLoC extends StatelessWidget {
  JokeBLoC jokeBLoC;
  @override
  Widget build(BuildContext context) {
    //获取到BLoC
    jokeBLoC = BLoCProvider.of<JokeBLoC>(context);
    ScrollController _scrollController = new ScrollController();
    _scrollController.addListener(() {
      if (_scrollController.position.pixels ==
          _scrollController.position.maxScrollExtent) {
        //滑动监听，通知BLoC加载下一页
        jokeBLoC.inJokesIndex.add(true);
      }
    });
    //页面创建时，开始加载数据
    jokeBLoC.inJokesIndex.add(false);

    return RefreshIndicator(
      onRefresh: () => refresh(),
      child: StreamBuilder<List<JokeBean>>(
        //监听JokeBLoC中的加载数据的流信息
          stream: jokeBLoC.outResultList,
          builder:
              (BuildContext context, AsyncSnapshot<List<JokeBean>> snapshot) {
                //当数据是空的时候，显示加载动画，这里有点问题：当请求出错时没有进行处理，会一直显示动画
            if (snapshot.data == null || snapshot.data.isEmpty) {
              return SpinKitWave(
                  color: Colors.redAccent, type: SpinKitWaveType.start);
            }
						//构建列表
            return ListView.builder(
                controller: _scrollController,
                itemCount: snapshot.data == null ? 1 : snapshot.data.length + 1,
                itemBuilder: (BuildContext context, int position) {
                  return _getRow(context, snapshot.data, position);
                });
          }),
    );
  }

  Future<void> refresh() async {
    jokeBLoC.inJokesIndex.add(false);
  }
}
```



这是一个大佬写的https://github.com/boeledi/Streams-Block-Reactive-Programming-in-Flutter

想要跑起来需要自己去申请一个key，具体看`tmdb_api.dart`

这是我写的 https://github.com/huangyuanlove/JanDan_flutter

**不要过度设计你的代码**

----

以上

