---
title: riverpod1.0+简介
tags: [Android,Flutter]
date: 2022-01-04 23:47:43
keywords:  [状态共享,状态管理,provider,riverpod]
---

Flutter更新到2.8了，最近打算重拾一下flutter，写点东西练练手。大家都清楚在flutter中状态管理确实挺麻烦的，从一开始的BLoC到provide、Provider，还有getX、Riverpod等等各式各样的状态管理库，我个人倾向于使用riverpod，它更像一个状态管理库；而getX更像一个开发的框架，实在是太大了:当你使用getX的时候，你是在用getX而不是flutter写应用。
<!--more-->

## 引入riverpod
demo中没有包含flutter_hook,所以我们选择引入flutter_riverpod即可
``` yaml
environment:
  sdk: ">=2.15.1 <3.0.0"
  flutter: ">=2.0.0"

dependencies:
  flutter:
    sdk: flutter
  flutter_riverpod: ^1.0.3
```

## hello world
首先，我们需要使用`ProviderScope`来包裹整个应用，也就是在main方法中
``` dart
void main() {
  runApp(ProviderScope(child: Home()));
}
```
然后我们可以声明一个Provider。一般情况下，我们会把各种各样的provider作为全局变量来引用，声明一个provider和声明一个函数没有多大的区别。
``` dart
final helloWorldProvider = Provider((_) => 'Hello world');
```
最后我们就可以读取Provider中的数据了。
在1.0.0之后的版本中，ConsumerWidget的build方法中提供了`WidgetRef`对象，用来取代0.14版本中的`useProvider`
``` dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

final helloWorldProvider  = Provider((_)=>"hello world");

void main() {
  runApp(const ProviderScope(child: Home()));
}

class Home extends ConsumerWidget{
  const Home({Key? key}) : super(key: key);

  @override
  Widget build(BuildContext context, WidgetRef ref) {

    final String value = ref.watch(helloWorldProvider);

    return MaterialApp(
      home: Scaffold(
        appBar: AppBar(title: const Text("riverpod demo"),),
        body: Center(
          child: Text(value),
        ),
      ),
    );
  }
}
```

## Provider

### 各种各样的Provider

具体可以看这里，https://pub.dev/documentation/flutter_riverpod/latest/flutter_riverpod/flutter_riverpod-library.html

下面列举了一些常用的Provider类型

* Provider

  https://pub.dev/documentation/riverpod/latest/riverpod/Provider-class.html


* StateProvider

https://pub.dev/documentation/riverpod/latest/riverpod/StateProvider-class.html

* StateNotifierProvider
  https://pub.dev/documentation/riverpod/latest/riverpod/StateNotifierProvider-class.html

* FutureProvider
  https://pub.dev/documentation/riverpod/latest/riverpod/FutureProvider-class.html

* StreamProvider
  https://pub.dev/documentation/riverpod/latest/riverpod/StreamProvider-class.html

  

### Provider的修饰符
#### .family

##### 使用

该修饰符适用于适用外部数据来构建provider的情况

一些常用情况

* 和[FutureProvider](https://pub.dev/documentation/riverpod/latest/riverpod/FutureProvider-class.html) 组合，来根据id获取消息
* 把当前Locale对象传给provider，用来进行国际化
* 在不访问对方属性的前提下连接两个provider

在使用family时，会额外的向provider提供一个属性，在provider中我们可以自由的使用该属性来创建某些状态

``` dart
final messagesFamily = FutureProvider.family<Message, String>((ref, id) async {
  return dio.get('http://my_api.dev/messages/$id');
});
```

这种情况下在使用`messagesFamily`时会有点语法上的变化，我们需要额外提供一个参数

``` dart
Widget build(BuildContext context, WidgetRef ref) {
  final response = ref.watch(messagesFamily('id'));
}
```

它还支持同时获取不同的属性

``` dart
@override
Widget build(BuildContext context, WidgetRef ref) {
  final frenchTitle = ref.watch(titleFamily(const Locale('fr')));
  final englishTitle = ref.watch(titleFamily(const Locale('en')));

  return Text('fr: $frenchTitle en: $englishTitle');
}
```

##### 参数限制

参数不限制类型，但必须实现`==`和`hashCode`两个方法；

如果参数不是constant的，比如我们想将输入框内容传给Provider，但是输入框的内容会变化的特别频繁并且不能复用，这种情况可能会导致内存泄露，可以使用`.autoDispose`修饰符来修复这个问题

``` dart
final characters = FutureProvider.autoDispose.family<List<Character>, String>((ref, filter) async {
  return fetchCharacters(filter: filter);
});
```

##### 传递多个参数

.family修饰符并没有内置提供过个参数的方法，另外一方面，这个参数可以是任意符合上面提到的限制的类型。
比如

* 元组
* 使用 Freezed 或 built_value 生成的对象
* 使用 equatable 的对象



** freezed **

``` dart
@freezed
abstract class MyParameter with _$MyParameter {
  factory MyParameter({
    required int userId,
    required Locale locale,
  }) = _MyParameter;
}

final exampleProvider = Provider.autoDispose.family<Something, MyParameter>((ref, myParameter) {
  print(myParameter.userId);
  print(myParameter.locale);
  // Do something with userId/locale
});

@override
Widget build(BuildContext context, WidgetRef ref) {
  int userId; // Read the user ID from somewhere
  final locale = Localizations.localeOf(context);

  final something = ref.watch(
    exampleProvider(MyParameter(userId: userId, locale: locale)),
  );

  ...
}
```

**Equatable**

``` dart
class MyParameter extends Equatable  {
  MyParameter({
    required this.userId,
    required this.locale,
  });

  final int userId;
  final Locale locale;

  @override
  List<Object> get props => [userId, locale];
}

final exampleProvider = Provider.family<Something, MyParameter>((ref, myParameter) {
  print(myParameter.userId);
  print(myParameter.locale);
  // Do something with userId/locale
});

@override
Widget build(BuildContext context, WidgetRef ref) {
  int userId; // Read the user ID from somewhere
  final locale = Localizations.localeOf(context);

  final something = ref.watch(
    exampleProvider(MyParameter(userId: userId, locale: locale)),
  );

  ...
}
```





#### .autoDispose

一个通用场景是能够自动释放长时间不适用Provider；

有很多个让我们这么做得理由，比如：

* 在使用Firebase时，关闭连接避免不必要的开销
* 当用户离开页面再进入页面时重置状态

我们可以使用内嵌的`.autoDispose`修饰符来支持上述场景

#####  使用

想要告诉Riverpod在不再使用provider时将其销毁，只需要在Provider之前加上`.autoDispose`即可

``` dart
final userProvider = StreamProvider.autoDispose<User>((ref) {

});
```

就这样，当`userProvider`不再使用时将会被自动销毁

注意通用参数是如何在autoDispose之后而不是之前传递的--autoDispose不是一个命名的构造函数。

当然，上面也提到可以和其他修饰符一起

``` dart
final userProvider = StreamProvider.autoDispose.family<User, String>((ref, id) {

});
```

##### ref.maintainState

用`autoDispose`标记一个提供者，也会在ref上增加一个额外的属性： `maintainState`。

该属性是一个布尔值（默认为false），允许提供者告诉Riverpod即使不再被监听，是否应该保留提供者的状态。

一个用例是在一个HTTP请求完成后，将这个标志设置为true:

``` dart
final myProvider = FutureProvider.autoDispose((ref) async {
  final response = await dio.get(...);
  ref.maintainState = true;
  return response;
});
```

这样，如果请求失败，用户离开屏幕后又重新进入，那么请求将被再次执行。但如果请求成功完成，状态将被保留，重新进入屏幕将不会触发新的请求。

##### 示例：取消http请求

autoDispose修改器可以与FutureProvider和ref.onDispose相结合，以便在不再需要HTTP请求时轻松取消。

要求：

* 当用户进入一个屏幕时，启动一个HTTP请求
* 如果用户在请求完成前离开屏幕，则取消HTTP请求
* 如果请求成功，离开并重新进入屏幕不会启动一个新的请求

``` dart
final myProvider = FutureProvider.autoDispose((ref) async {
  // An object from package:dio that allows cancelling http requests
  final cancelToken = CancelToken();
  // When the provider is destroyed, cancel the http request
  ref.onDispose(() => cancelToken.cancel());

  // Fetch our data and pass our `cancelToken` for cancellation to work
  final response = await dio.get('path', cancelToken: cancelToken);
  // If the request completed successfully, keep the state
  ref.maintainState = true;
  return response;
});
```

##### 参数类型'AutoDisposeProvider'不能分配给参数类型'AlwaysAliveProviderBase'。

当使用.autoDispose时，你可能会发现自己的应用程序无法编译，出现类似的错误。

> The argument type 'AutoDisposeProvider' can't be assigned to the parameter type 'AlwaysAliveProviderBase'

可能是因为你试图在一个没有标记为.autoDispose的提供者中监听一个标记为.autoDispose的提供者，例如：

``` dart
final firstProvider = Provider.autoDispose((ref) => 0);

final secondProvider = Provider((ref) {
  // The argument type 'AutoDisposeProvider<int>' can't be assigned to the
  // parameter type 'AlwaysAliveProviderBase<Object, Null>'
  ref.watch(firstProvider);
});
```

这是不可取的，因为它将导致firstProvider永远不会被dispose。我们可以考虑将 `secondProvider` 标记为 `.autoDispose来修复这个问题：

``` dart
final firstProvider = Provider.autoDispose((ref) => 0);

final secondProvider = Provider.autoDispose((ref) {
  ref.watch(firstProvider);
});
```



## WidgetRef

### 获取WidgetRef对象

#### 从其他Provider对象中获取

``` dart
final provider = Provider((ref) {
  // use ref to obtain other providers
  final repository = ref.watch(repositoryProvider);
  return SomeValue(repository);
})
```

`ref`对象可以很安全的在provider之间传递，一个常见的用法就是讲`ref`传递给 [StateNotifier](https://pub.dev/documentation/state_notifier/latest/state_notifier/StateNotifier-class.html)

``` dart
final counter = StateNotifierProvider<Counter, int>((ref) {
  return Counter(ref);
});

class Counter extends StateNotifier<int> {
  Counter(this.ref): super(0);

  final Ref ref;

  void increment() {
    // Counter can use the "ref" to read other providers
    final repository = ref.read(repositoryProvider);
    repository.post('...');
  }
}
```

这么做可以让Counter内部读取provider状态

#### 从Widget对象中获取ref

一般情况下Widget对象中是没有ref对象中，但riverpod提供了几种解决方案

* 使用ConsumerWidget替换StatelessWidget

ConsumerWidget和StatelessWidget基本相同(虽然是继承了StatefulWidget)，只是在build方法中多了一个WidgetRef对象

``` dart
class HomeView extends ConsumerWidget {
  const HomeView({Key? key}): super(key: key);

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    // use ref to listen to a provider
    final counter = ref.watch(counterProvider);
    return Text('$counter');
  }
}
```

* 使用ConsumerStatefulWidget+ConsumerState 替换 StatefulWidget+State

``` dart
class HomeView extends ConsumerStatefulWidget {
  const HomeView({Key? key}): super(key: key);

  @override
  HomeViewState createState() => HomeViewState();
}

class HomeViewState extends ConsumerState<HomeView> {
  @override
  void initState() {
    super.initState();
    // "ref" can be used in all life-cycles of a StatefulWidget.
    ref.read(counterProvider);
  }

  @override
  Widget build(BuildContext context) {
    // We can also use "ref" to listen to a provider inside the build method
    final counter = ref.watch(counterProvider);
    return Text('$counter');
  }
}

```

* 使用 HookConsumerWidget 替换 HookWidget

``` dart
class HomeView extends HookConsumerWidget {
  const HomeView({Key? key}): super(key: key);

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    // HookConsumerWidget allows using hooks inside the build method
    final state = useState(0);

    // We can also use the ref parameter to listen to providers.
    final counter = ref.watch(counterProvider);
    return Text('$counter');
  }
}
```





### WidgetRef对象的方法


这里的`WidgetRef`对象在读取Provider中的数据时，提供了`read`、`listen`和`watch`方法。至于什么情况下选用哪个方法，这里有三个建议
> * 当我们需要监听变化并且从Provider中获取数据时，比如当数据变化时我们需要重新构建Widget，这时我们可以使用`ref.watch`
> * 当我们需要监听变化去执行某个动作时，我们可以使用`ref.listen`
> * 当我们仅需要读取数据不关心数据的变化时，比如点击某个按钮时，根据状态来判断下一步动作时，我们可以使用`ref.read`



* ref.watch
``` dart
final counterProvider = StateProvider((_)=> 0);
class Home extends ConsumerWidget{
  const Home({Key? key}) : super(key: key);
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final int count = ref.watch(counterProvider);
    return MaterialApp(
      home: Scaffold(
        appBar: AppBar(title: const Text("riverpod demo"),),
        body: Center(
          child: Column(
            children: [
              Text('$count')
            ],
          ),
        ),
        floatingActionButton: FloatingActionButton(onPressed: ()=>{
          ref.read(counterProvider.state).state++
        },child: const Text("点击"),),
      ),
    );
  }
}
```
* ref.read

使用该方法可以没有任何影响的获取一次provider的状态，但是作者提示我们尽量不要使用该方法，它只是用来解决使用`watch|listen`不方便的问题，如果可以，尽量使用`watch|listen.`这里有个使用read方法的示例https://riverpod.dev/docs/concepts/combining_providers#can-i-read-a-provider-without-listening-to-it

``` dart
final counterProvider = StateNotifierProvider<Counter, int>((ref) => Counter());

class HomeView extends ConsumerWidget {
  const HomeView({Key? key}): super(key: key);

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    return Scaffold(
      floatingActionButton: FloatingActionButton(
        onPressed: () {
          // Call `increment()` on the `Counter` class
          ref.read(counterProvider.notifier).increment();
        },
      ),
    );
  }
}
```



* ref.listen

和`ref.watch`相似，我们也可以使用`ref.listen`来观察provider。他们的区别就是当provider状态变化时，我们可以调用自己定义的方法。该方法需要两个参数，第一个参数是要监听的provider对象，第二个参数是回调方法，

``` dart
final counterProvider = StateNotifierProvider<Counter, int>((ref) => Counter());

final anotherProvider = Provider((ref) {
  ref.listen<int>(counterProvider, (int? previousCount, int newCount) {
    print('The counter changed ${newCount}');
  });
  ...
});
```

或者

``` dart
final counterProvider = StateNotifierProvider<Counter, int>((ref) => Counter());

class HomeView extends ConsumerWidget {
  const HomeView({Key? key}): super(key: key);

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    ref.listen<int>(counterProvider, (int? previousCount, int newCount) {
      print('The counter changed ${newCount}');
    });
    ...
  }
}
```

### 决定订阅什么

比如我们有一个StreamProvider

``` dart
final userProvider = StreamProvider<User>(...);
```

我们可以这么去订阅

* 通过监听provider本身来同步获取当前状态

``` dart
Widget build(BuildContext context, WidgetRef ref) {
  AsyncValue<User> user = ref.watch(userProvider);

  return user.when(
    loading: () => const CircularProgressIndicator(),
    error: (error, stack) => const Text('Oops'),
    data: (user) => Text(user.name),
  );
}
```

* 通过监听`userProvider.stream`来获取对应的stream

``` dart
Widget build(BuildContext context, WidgetRef ref) {
  Stream<User> user = ref.watch(userProvider.stream);
}
```

* 通过监听`userProvider.future`来获取一个能得到最新状态的Future

``` dart
Widget build(BuildContext context, WidgetRef ref) {
  Future<User> user = ref.watch(userProvider.future);
}
```

### 使用"select" 来决定哪些值变化时进行重建

比如我们有一个User对象

``` dart
abstract class User {
  String get name;
  int get age;
}
```

但是我们在渲染页面时只用到了name属性

``` dart
Widget build(BuildContext context, WidgetRef ref) {
  User user = ref.watch(userProvider);
  return Text(user.name);
}
```

这种情况下，如果`age`属性发生了变化，该Widget就会重建，显然这不是我们想要的。这时候我们可以使用`select`来选择对象的某些属性来监听

``` dart
Widget build(BuildContext context, WidgetRef ref) {
  String name = ref.watch(userProvider.select((user) => user.name))
  return Text(name);
}
```

当然，`select`同样适用于`listen`方法

``` dart
ref.listen<String>(
  userProvider.select((user) => user.name),
  (String? previousName, String newName) {
    print('The user name changed $newName');
  }
);
```

需要注意的是，这里没必要一定返回对象的属性，只要复写了`==`的值都可以正常工作，比如

``` dart
final label = ref.watch(userProvider.select((user) => 'Mr ${user.name}'));

````
