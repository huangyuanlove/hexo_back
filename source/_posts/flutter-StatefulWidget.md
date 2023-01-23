---
title: flutter_StatefulWidget
tags: [Flutter]
date: 2023-01-18 16:00:47
keywords: [flutter,statefulWidget,State]
---
### createState()是何时被调用的？
断点查看调用栈，发现是在`StatefulElement`的构造方法中创建的,而`element`的创建则是在父元素调用`inflateWidget`时触发子元素的`createElement`方法创建的
``` dart
  StatefulElement(StatefulWidget widget)
      : _state = widget.createState(),
        super(widget) {
    state._element = this;
    state._widget = widget;
  }
```
去掉断言代码可以看到，在构造方法中调用了`createState()`来创建`State`对象，接着对`_state`对象的`_element`和`_widget`成员进行赋值。
到这里我们可以清楚的知道:`State`和`StatefulElement`互相持有，并且两者都持有`StatefulWidget`。

### State类中的方法

这里面定义了生命周期方法
* initState()
* didUpdateWidget(covariant T oldWidget)
* void reassemble()
* void deactivate()
* void activate()
* void dispose()
* Widget build(BuildContext context)
* void didChangeDependencies()

其实看一下这些方法上面的注释基本上就能理解的差不多，断点走一遍流程，也就都了解了

### 回调时机

`StatefulElement`继承自`ComponentElement`类，该类在`mount()`时调用的了`_firstBuild()`方法，这个方法被`StatefulElement`覆写，可以看到在这里里面调用了`state.initState()`方法
``` dart 
  @override
  void _firstBuild() {
    try {
      _debugSetAllowIgnoredCallsToMarkNeedsBuild(true);
      final Object? debugCheckForReturnedFuture = state.initState() as dynamic;
    } finally {
      _debugSetAllowIgnoredCallsToMarkNeedsBuild(false);
    }
    state.didChangeDependencies();
    super._firstBuild();
  }
```
紧着这就调用了`state.didChangeDependencies()`方法,最后调用了`super._firstBuild()`;
还是在`ComponentElement`类中的`_firstBuild()`方法中调用了`rebuild()-->performRebuild`,这里`performRebuild()`在`StatefulElement`有被重写
``` dart 
  @override
  void performRebuild() {
    if (_didChangeDependencies) {
      state.didChangeDependencies();
      _didChangeDependencies = false;
    }
    super.performRebuild();
  }
```
这里的`_didChangeDependencies`默认为`false`，所以第一次进来并不会触发`state.didChangeDependencies()`方法;接下来执行了`super.performRebuild()`;
同样的在`ComponentElement`类中的`performRebuild()`方法中调用了`build()`方法,当然这里的`build`方法已经被子类`StatefulElement`重写，调用了`state.build(this)`方法，然后调用了`updateChild()`方法
``` dart 
  void performRebuild() {
    assert(_debugSetAllowIgnoredCallsToMarkNeedsBuild(true));
    Widget? built;
    try {
      built = build();
      debugWidgetBuilderValue(widget, built);
    } catch (e, stack) {
      _debugDoingBuild = false;
      built = ErrorWidget.builder(...);
    } finally {
       _dirty = false;
      
    }
    try {
      _child = updateChild(_child, built, slot);
      
    } catch (e, stack) {
      built = ErrorWidget.builder();
      _child = updateChild(null, built, slot);
    }
  }
```
这里就接上了控件是如何进行挂载的

### 如何更新

我们知道在`StatefulWidget`中可以使用`setState()`来更新页面内容，那么表示状态的属性是在什么时机赋值，这里有两种方式
``` dart
_color = Colors.red;
setState((){
});
```
或者
``` dart 
setStaate((){
  _color = Colors.red;
});
```
戳进去看源码
``` dart 
void setState(VoidCallback fn) {
  assert(fn != null);
  final Object? result = fn() as dynamic;
  assert(() {
    if (result is Future) {
      throw FlutterError.fromParts(<DiagnosticsNode>[
        ErrorSummary('setState() callback argument returned a Future.'),
        ErrorDescription(
          'The setState() method on $this was called with a closure or method that '
          'returned a Future. Maybe it is marked as "async".',
        ),
        ErrorHint(
          'Instead of performing asynchronous work inside a call to setState(), first '
          'execute the work (without updating the widget state), and then synchronously '
          'update the state inside a call to setState().',
        ),
      ]);
    }
    // We ignore other types of return values so that you can do things like:
    //   setState(() => x = 3);
    return true;
  }());
  _element!.markNeedsBuild();
}
```
一坨断言，判断`callback`是不是空，`callback`的返回值是不是`Future`类型;然后调用`_element!.markNeedsBuild()`。所以就这段代码来看，上面两种写法都可以，但还是建议向源码看齐：状态属性的改变写在`callback`中，确保在`markNeedsBuild()`之前，状态值是自己期望的结果；

### 更新
``` dart 
  void markNeedsBuild() {
    if (_lifecycleState != _ElementLifecycle.active)
      return;
    if (dirty)
      return;
    _dirty = true;
    owner!.scheduleBuildFor(this);
  }
```
如果当前状态不是active则不标记，如果已经标记过也不在标记；
``` dart 
  /// Adds an element to the dirty elements list so that it will be rebuilt
  /// when [WidgetsBinding.drawFrame] calls [buildScope].
  void scheduleBuildFor(Element element) {

    if (element._inDirtyList) {
      _dirtyElementsNeedsResorting = true;
      return;
    }
    if (!_scheduledFlushDirtyElements && onBuildScheduled != null) {
      _scheduledFlushDirtyElements = true;
      onBuildScheduled!();
    }
    _dirtyElements.add(element);
    element._inDirtyList = true;
  }
```
这里也是进行了二次判断，保证不会被多次重绘。注意`onBuildScheduled()`的调用，它的定义是`VoidCallback? onBuildScheduled`;并且是在`BuildOwner`类中的构造方法中初始化的，那么这个`onBuildScheduled`到底是什么方法?断点看一下是`WidgetsBinding#_handleBuildScheduled`这个方法。它是在什么时候被赋值的？`owner`是`element`对象中的一个成员变量`_owner`,搜一下看一看到是在`mount()`方法中赋值的，值为`parent.owner`。还记的之前初始化根节点的的时候调用的`WidgetsBinding#attachRootWidget(Widget rootWidget)`这个方法中创建`RenderObjectToWidgetAdapter`对象后调用的`attachToRenderTree()`方法中有传入`BuildOwner`对象，接着向上查找，发现是在`initInstances()`方法中创建的.
``` dart
  void initInstances() {
    super.initInstances();
    _instance = this;
    // Initialization of [_buildOwner] has to be done after
    // [super.initInstances] is called, as it requires [ServicesBinding] to
    // properly setup the [defaultBinaryMessenger] instance.
    _buildOwner = BuildOwner();
    buildOwner!.onBuildScheduled = _handleBuildScheduled;
    platformDispatcher.onLocaleChanged = handleLocaleChanged;
    platformDispatcher.onAccessibilityFeaturesChanged = handleAccessibilityFeaturesChanged;
    SystemChannels.navigation.setMethodCallHandler(_handleNavigationInvocation);

    platformMenuDelegate = DefaultPlatformMenuDelegate();
  }
```
在`_handleBuildScheduled` 中就只是调用了`ensureVisualUpdate()`方法，然后调用了`scheduleFrame()`;
``` dart
  void scheduleFrame() {
    if (_hasScheduledFrame || !framesEnabled)
      return;
    ensureFrameCallbacksRegistered();
    platformDispatcher.scheduleFrame();
    _hasScheduledFrame = true;
  }
```
这里先确保两个回调确实被注册了，然后通过`platformDispatcher.scheduleFrame()`这个*native*方法向系统发送一个帧调度的请求。
然后会回调`_handleDrawFrame()`方法，也就是`ensureFrameCallbacksRegistered()`方法中确保两个回调方法
``` dart
  void handleDrawFrame() {
    assert(_schedulerPhase == SchedulerPhase.midFrameMicrotasks);
    _frameTimelineTask?.finish(); // end the "Animate" phase
    try {
      // PERSISTENT FRAME CALLBACKS
      _schedulerPhase = SchedulerPhase.persistentCallbacks;
      for (final FrameCallback callback in _persistentCallbacks)
        _invokeFrameCallback(callback, _currentFrameTimeStamp!);

      // POST-FRAME CALLBACKS
      _schedulerPhase = SchedulerPhase.postFrameCallbacks;
      final List<FrameCallback> localPostFrameCallbacks =
          List<FrameCallback>.of(_postFrameCallbacks);
      _postFrameCallbacks.clear();
      for (final FrameCallback callback in localPostFrameCallbacks)
        _invokeFrameCallback(callback, _currentFrameTimeStamp!);
    } finally {
      _schedulerPhase = SchedulerPhase.idle;
      _frameTimelineTask?.finish(); // end the Frame
      assert(() {
        if (debugPrintEndFrameBanner)
          debugPrint('▀' * _debugBanner!.length);
        _debugBanner = null;
        return true;
      }());
      _currentFrameTimeStamp = null;
    }
  }
```
然后是调用 `_invokeFrameCallback`，这里的`_persistentCallbacks`是个`list`，通过
``` dart
void addPersistentFrameCallback(FrameCallback callback) {
  _persistentCallbacks.add(callback);
}
```
这个方法添加回调；这里的回调是在`RendererBinding`类中的`initInstances()`方法中注册的，实际上调用的方法是
``` dart
_handlePersistentFrameCallback
  void _handlePersistentFrameCallback(Duration timeStamp) {
    drawFrame();
    _scheduleMouseTrackerUpdate();
  }
```
然后调用了`drawFrame();`方法,需要注意的是:`RendererBinding`是一个*mixin*的类，被`WidgetsBinding`混入，并且`WidgetsBinding`类中重写了`drawFrame()`方法，所以最后走的是`WidgetsBinding`类中的`drawFrame()`方法；在这里面调用了`buildOwner!.buildScope(renderViewElement!);`
在这个方法中先对脏列表进行排序
``` dart
  static int _sort(Element a, Element b) {
    if (a.depth < b.depth)
      return -1;
    if (b.depth < a.depth)
      return 1;
    if (b.dirty && !a.dirty)
      return -1;
    if (a.dirty && !b.dirty)
      return 1;
    return 0;
  }
```
然后循环调用`element.rebuild();`触发`performRebuild()`接着就是`widget`的`build()`方法被触发.
