---
title: riverpod2中的修饰符及注解
tags: [Flutter,riverpod]
date: 2024-03-27 10:43:24
keywords: riverpod,flutter,Provider,ConsumerWidget,StateProvider,StateNotifierProvider,FutureProvider,ChangeNotifierProvider,NotifierProvider,AsyncNotifierProvider
---

原文来自：https://codewithandrea.com/articles/flutter-state-management-riverpod/


### 监听、观察、选择、记录、重建   

#### ref.read or ref.watch 

在上一篇中，我们使用了 ref.read和 ref.watch，那么这两者在什么情况下使用？
一般来讲，如果需要在Provider 值发生改变时更新依赖它的 widget，就使用 watch。比如 我们在 build 函数中使用watch,这确保了如果提供者的值发生更改，我们会重建依赖它的widget.
``` dart
final counterStateProvider = StateProvider<int>((_) => 0);
  class CounterWidget extends ConsumerWidget {
    @override
    Widget build(BuildContext context, WidgetRef ref) {
      // 1. watch the provider and rebuild when the value changes
      final counter = ref.watch(counterStateProvider);
      return ElevatedButton(
        // 2. use the value
        child: Text('Value: $counter'),
        // 3. change the state inside a button callback
        onPressed: () => ref.read(counterStateProvider.notifier).state++,
      );
    }
  }
```
如果我们只是读取到 Provider 的值做操作，一般使用 read，比如上面点击 ElevatedButton 时我们获取到 notifier 之后对count 做了++操作
需要注意的是：notifier语法仅适用于StateProvider和StateNotifierProvider，其工作方式如下：
- 在StateProvider上调用ref.read(provider.notifier)，以返回底层的StateController，我们可以使用它来修改状态。
- 在StateNotifierProvider上调用ref.read(provider.notifier)，以返回底层的StateNotifier，以便我们可以调用其方法。

#### select
我们有时候也会遇到这种需求：一个对象有多个属性，希望只有特定属性发生变化时才更新页面，这时候我们就可以使用 select 来完成
``` dart 
class Student {
  Student(this.firstName, this.lastName, this.age);

  String firstName;
  String lastName;
  int age;
  Student copy() {
    return new Student(firstName, lastName, age);
  }
}
```
我们只需要在年龄发生改变时更新页面
``` dart
Widget build(BuildContext context, WidgetRef ref) {
    final int age = ref.watch( provider.select((value) => value.age) );
    return Text("$age")
}
```


#### listen

除此之外，我们还有 listen， 比如我们希望在 Provider 内容发生变化时弹出一个 SnakeBar提示用户，我们可以这样做:
``` dart
final counterStateProvider = StateProvider<int>((_) => 0);

class CounterWidget extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    // if we use a StateProvider<T>, the type of the previous and current 
    // values is StateController<T>
    ref.listen<StateController<int>>(counterStateProvider.state, (previous, current) {
      // note: this callback executes when the provider value changes,
      // not when the build method is called
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text('Value is ${current.state}')),
      );
    });
    // watch the provider and rebuild when the value changes
    final counter = ref.watch(counterStateProvider);
    return ElevatedButton(
      // use the value
      child: Text('Value: $counter'),
      // change the state inside a button callback
      onPressed: () => ref.read(counterStateProvider.notifier).state++,
    );
  }
}
```

#### 日志
我们需要在 Provider 内容发生改变时打印日志，以方便我们进行调试，但又不想在每个 Provider 中都加上打印代码，应该怎么办？
在 Rivepod 中有一个ProviderObserver类，我们可以继承它实现自己的逻辑
``` dart
class ProviderLogger extends ProviderObserver {
    @override
    void didUpdateProvider(
      ProviderBase provider,
      Object? previousValue,
      Object? newValue,
      ProviderContainer container,
    ) {
      print('[${provider.name ?? provider.runtimeType}] value: $newValue');
    }
  }
```
同样的，需要将它添加到 ProviderScope内的观察者列表中
``` dart
void main() {
    runApp(
      ProviderScope(observers: [ProviderLogger()], child: MyApp()),
    );
  }
```
为了方便我们区分是哪一个 Provider 的日志，我们在声明 Provider 时可以提供一个名字
``` dart 
final counterStateProvider = StateProvider<int>((ref) {
    return 0;
  }, name: 'main_page_counter');
```




### 修饰符

#### autoDispose
上一篇中提到Riverpod 作者强烈建议将 Provider 声明为全局的而不是类内部，这样有一个问题，页面销毁时也就是provider没有任何监听者时并不会被重置。再次进入页面后还是页面销毁之前的值，比如 StateProvider 实现的计数功能。当我们在页面中点击加号，将数字变为 5 后，返回上一个页面后再次打开该页面，会发现页面还是 5。这有时不符合我们的需求，这里我们可以使用autoDispose来实现：
``` dart
final autoDisposeProvider = StateProvider.autoDispose<int>(((ref) {
  ref.onDispose(() {
    debugPrint("countProvider onDispose");
  });

  return 1;
}));

final countProvider = StateProvider<int>((ref) => 1);


class RiverpodAutoDisposeWidget extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final autoDisposeValue = ref.watch(autoDisposeProvider);
    final normalValue = ref.watch(countProvider);
    return Scaffold(
      appBar: AppBar(
        title: Text("AutoDispose"),
        centerTitle: true,
      ),
      body: Column(children: [
        Text("autoDispose:当privder不被使用时，可以被自动释放"),
        Text("autoDisposeValue-> ${autoDisposeValue}"),
        Text("normalValue-> ${normalValue}"),
        Row(
          children: [
            ElevatedButton(
                onPressed: (() {
                  ref.read(autoDisposeProvider.notifier).state++;
                  ref.read(countProvider.notifier).state++;
                }),
                child: Text("增加计数"))
          ],
        )
      ]),
    );
  }
}


```
可以看到当页面被销毁时，被autoDispose修饰的 provider 会被释放并重置。这里的 ref.onDispose会在没有监听者之后调用。

#### keepAlive

我们可以使用 ref.keepAlive实现超时缓存。
简单粗暴点就是这样：
``` dart
final tmpProvider = StateProvider.autoDispose<int>((ref){

    // get the [KeepAliveLink]
    final link = ref.keepAlive();
    // start a 30 second timer
    final timer = Timer(const Duration(seconds: 30), () {
      // dispose on timeout
      link.close();
    });
    // make sure to cancel the timer when the provider state is disposed
    // (prevents undesired test failures)
    ref.onDispose(() => timer.cancel());

  return 1;
});

```
我们可以使用`extension`封装一下
``` dart
extension AutoDisposeRefCache on AutoDisposeRef {
  void cacheFor(Duration duration) {
    final link = keepAlive();
    final timer = Timer(duration, () {
      link.close();
    });
    onDispose(() {
      timer.cancel();
    });
  }
}

//使用
final timerCachedProvider = Provider.autoDispose<int>((ref) {
  ref.cacheFor(Duration(minutes: 5));
  return 1;
});
``` 
#### family
我们可以用它向 Provider 提供参数，比如我们的计数器示例，比如想从指定的数字开始
``` dart
final countProviderBase = StateProvider.autoDispose.family<int,int> ((ref,start){
  return start;
});
//使用
final startValue =  ref.watch(countProviderBase(10));
```

#### 依赖覆盖
有时候我们希望使用 Provider 存储无法立即获取的对象或者值，比如做本地存储时用的`shared_preferences`。但是它的初始化是异步的，如果我们直接在 Provider 中使用则会提示
``` dart
final spProvider = Provider<SharedPreferences>((ref) {
  return SharedPreferences.getInstance();//不可以这样使用
  //The return type 'Future<SharedPreferences>' isn't a 'SharedPreferences', as required by the closure's context.
});
```
这时候我们可以先抛出一个为实现的异常
``` dart
final sharedPreferencesProvider = Provider<SharedPreferences>((ref) {
    throw UnimplementedError();
  });
```
然后我们可以在 ProviderScope 组件中进行依赖覆盖
``` dart
  Future<void> main() async {
    WidgetsFlutterBinding.ensureInitialized();
    final sharedPreferences = await SharedPreferences.getInstance();
    runApp(ProviderScope(
      overrides: [
        // override the previous value with the new object
        sharedPreferencesProvider.overrideWithValue(sharedPreferences),
      ],
      child: MyApp(),
    ));
  }
```
这样我们就可以在任何地方观察sharedPreferencesProvider对象，而无需使用基于Future的API

### 注解
我们在使用 Provider 时都需要手动编写，我们是否可以使用 build_runner生成？Riverpod 中已经提供了这种方式，但当前只支持以下几种
- Provider
- FutureProvider
- StreamProvider
- NotifierProvider
- AsyncNotifierProvider
  
#### 添加依赖

``` yaml
dependencies:
  # or flutter_riverpod/hooks_riverpod as per https://riverpod.dev/docs/getting_started
  riverpod: ^2.4.10
  # the annotation package containing @riverpod
  riverpod_annotation: ^2.3.4
dev_dependencies:
  # a tool for running code generators
  build_runner:
  # the code generator
  riverpod_generator: ^2.3.9
  # riverpod_lint makes it easier to work with Riverpod
  riverpod_lint: ^2.3.7
  # import custom_lint too as riverpod_lint depends on it
  custom_lint:
```
其中`riverpod_lint`和`custom_lint`这两个包是可选的。
之后我们需要在`watch`模式下启动代码生成器
`flutter pub run build_runner watch -d`,较新的flutter版本会提示使用`dart run build_runner watch -d`。

#### 简单使用
我们来看个简单的示例，从 Provider 开始
没有使用注解生成器之前：
``` dart
// dio_provider.dart
import 'package:dio/dio.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
// a provider for the Dio client to be used by the rest of the app
final dioProvider = Provider<Dio>((ref) {
  return Dio();
});
```
使用注解生成器之后
``` dart
import 'package:dio/dio.dart';
// 1. import the riverpod_annotation package
import 'package:riverpod_annotation/riverpod_annotation.dart';
// 2. add a part file
part 'dio_provider.g.dart';


//需要执行 flutter pub run build_runner watch -d 来生成对应代码
@riverpod
Dio dio(DioRef ref){
  return Dio();
}
```
会生成`dio_provider.g.dart`文件，内容如下
``` dart
// GENERATED CODE - DO NOT MODIFY BY HAND

part of 'dio_provider.dart';

// **************************************************************************
// RiverpodGenerator
// **************************************************************************

String _$dioHash() => r'58eeefbd0832498ca2574c1fe69ed783c58d1d8f';

/// See also [dio].
@ProviderFor(dio)
final dioProvider = AutoDisposeProvider<Dio>.internal(
  dio,
  name: r'dioProvider',
  debugGetCreateSourceHash:
      const bool.fromEnvironment('dart.vm.product') ? null : _$dioHash,
  dependencies: null,
  allTransitiveDependencies: null,
);

typedef DioRef = AutoDisposeProviderRef<Dio>;
// ignore_for_file: type=lint
// ignore_for_file: subtype_of_sealed_class, invalid_use_of_internal_member, invalid_use_of_visible_for_testing_member

```

#### autoDispose和 keepAlive
在使用注解生成代码时，autoDispose 现在默认启用，并已重命名为 keepAlive。如果不想销毁 provider，可以将keepAlive设置为 true
``` dart
@Riverpod(keepAlive: true)
int counter(CounterRef ref){
  return 1;
}
```
生成的代码是这样的
``` dart
String _$counterHash() => r'6b26baf29ab7c65258c6367ad62133458d88a2b3';

/// See also [counter].
@ProviderFor(counter)
final counterProvider = Provider<int>.internal(
  counter,
  name: r'counterProvider',
  debugGetCreateSourceHash:
      const bool.fromEnvironment('dart.vm.product') ? null : _$counterHash,
  dependencies: null,
  allTransitiveDependencies: null,
);

typedef CounterRef = ProviderRef<int>;
// ignore_for_file: type=lint
// ignore_for_file: subtype_of_sealed_class, invalid_use_of_internal_member, invalid_use_of_visible_for_testing_member

```
可以看到，当 keepAlive 是默认值(true)时，使用的是`AutoDisposeProvider`,为 false 时，使用的是`Provider`

#### FutureProvider 和 StreamProvider
几乎是一样的写法

``` dart
@riverpod
Future<String> generateName(GenerateNameRef ref) async {
    await Future.delayed(Duration(seconds: 5));
    final wordPair = generateWordPairs().first;
    return "${wordPair.first}  ${wordPair.second}";
}

@riverpod
Stream<int> timeCount(TimeCountRef ref){
  return Stream.periodic(Duration(seconds: 1),(number){
    return number +1;
  });
}
``` 

#### NotifierProvider 

我们把上一篇中`NotifierProvider`例子简化一下，还是那个万能的计数器

``` dart
//声明一个Notifier对象
class Counter extends Notifier<int>{
  @override
  int build() {
   return 0;
  }

 //对 state 进行操作，也可以在外部直接操作
  void increment(){
    state ++;
  }

}
//两种 provider 的声明方式
final counterProvider = NotifierProvider<Counter, int>(() {
  return Counter();
});
// final counterProvider = NotifierProvider<Counter, int>(Counter.new);
```
在 widget 中使用
``` dart
class RiverpodGeneratorWidget extends ConsumerWidget{
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final count = ref.watch(counterProvider);
    return Scaffold(
      appBar: AppBar(title: Text("RiverpodGeneratorWidget"),centerTitle: true,),
      body: Column(children: [
        Text("count $count")
      ],),
      floatingActionButton: FloatingActionButton(onPressed: (){
        //调用 notifier 中定义的方法
        ref.read(counterProvider.notifier).increment();
        //直接获取到 state 进行操作
        ref.read(counterProvider.notifier).state++;
      },child: Icon(Icons.add),
      ),
    );
  }

}
```
那么我们如是用注解代码生成？
``` dart
//counter.dart
import 'package:riverpod_annotation/riverpod_annotation.dart';
part 'counter.g.dart';
@riverpod
class Counter extends _$Counter{
  @override
  int build() {
    return 0;
  }
  void increment(){
    state ++;
  }
}
```
需要注意的是，这里我们需要继承`_$Counter`而不是`Notifier`。
因为 `Counter`中的 `build` 方法返回值是`int`类型，生成的代码中也就使用了 `int`类型。简单来讲就是`build`方法的返回值类型决定了 `state` 的类型.

还是需要运行 `flutter pub run build_runner watch` 或者 `dart run build_runner watch`,这时会生成`counter.g.dart`文件，内容如下
``` dart
// GENERATED CODE - DO NOT MODIFY BY HAND

part of 'counter.dart';

// **************************************************************************
// RiverpodGenerator
// **************************************************************************

String _$counterHash() => r'7015b4a05f8ed24a914f6b3aad12be335d0c73d7';

/// See also [Counter].
@ProviderFor(Counter)
final counterProvider = AutoDisposeNotifierProvider<Counter, int>.internal(
  Counter.new,
  name: r'counterProvider',
  debugGetCreateSourceHash:
      const bool.fromEnvironment('dart.vm.product') ? null : _$counterHash,
  dependencies: null,
  allTransitiveDependencies: null,
);

typedef _$Counter = AutoDisposeNotifier<int>;
// ignore_for_file: type=lint
// ignore_for_file: subtype_of_sealed_class, invalid_use_of_internal_member, invalid_use_of_visible_for_testing_member

```

#### AsyncNotifierProvider
也是同样的
``` dart
@riverpod
class AsyncCounters extends _$AsyncCounters{
  @override
  FutureOr<int> build(){
    return Future.delayed(Duration(seconds: 3),(){
      return 1;
    });
  }
}
```
生成的代码如下
``` dart
// GENERATED CODE - DO NOT MODIFY BY HAND

part of 'async_counter.dart';

// **************************************************************************
// RiverpodGenerator
// **************************************************************************

String _$asyncCountersHash() => r'787b7c6513c7794fa310550d32594b97238e7e3c';

/// See also [AsyncCounters].
@ProviderFor(AsyncCounters)
final asyncCountersProvider =
    AutoDisposeAsyncNotifierProvider<AsyncCounters, int>.internal(
  AsyncCounters.new,
  name: r'asyncCountersProvider',
  debugGetCreateSourceHash: const bool.fromEnvironment('dart.vm.product')
      ? null
      : _$asyncCountersHash,
  dependencies: null,
  allTransitiveDependencies: null,
);

typedef _$AsyncCounters = AutoDisposeAsyncNotifier<int>;
// ignore_for_file: type=lint
// ignore_for_file: subtype_of_sealed_class, invalid_use_of_internal_member, invalid_use_of_visible_for_testing_member

```

### 其他
#### Notifier 和 AsyncNotifier：是否值得使用？
长时间以来，StateNotifier 一直在为我们提供服务，提供了一个存储复杂状态和修改状态逻辑的地方，使其不再依赖于小部件树。

Notifier 和 AsyncNotifier 旨在取代 StateNotifier 并带来一些新的好处：

更容易执行复杂的异步初始化
更符合人体工程学的 API：不再需要传递 ref
不再需要手动声明提供者（如果使用 Riverpod Generator）
对于新项目来说，这些好处是值得的，因为新的类可以帮助您用更少的代码实现更多的功能。

但如果您有很多现有代码使用 StateNotifier，则由您决定是否（或何时）迁移到新的语法。

无论如何，StateNotifier 还会存在一段时间，如果您愿意，可以逐个迁移您的提供者。
#### 使用 generator 还是手动编写 provider
使用 generator 需要我们执行额外的代码来生成对应的代码文件，并且在编写生成代码时体验不是那么的友好。但另外一方面，能省去我们编写模板的代码的时间。如何使用，看个人喜好。


