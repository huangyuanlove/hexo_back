---
title: flutter_runApp到挂载根节点
tags: [Flutter]
date: 2023-01-18 10:18:15
keywords: [Flutter,runApp,根节点的挂载]
---

## 入口

flutter应用的入口点在main方法中调用的`runApp(Widget app)`方法中



``` dart  widgets.binding.runApp
void runApp(Widget app) {
  WidgetsFlutterBinding.ensureInitialized()
    ..scheduleAttachRootWidget(app)
    ..scheduleWarmUpFrame();
}
```

这里的`WidgetsFlutterBinding`混入了七个 `xxxbinding`

* [GestureBinding], which implements the basics of hit testing.
* [SchedulerBinding], which introduces the concepts of frames.
* [ServicesBinding], which provides access to the plugin subsystem.
* [PaintingBinding], which enables decoding images.
* [SemanticsBinding], which supports accessibility.
* [RendererBinding], which handles the render tree.
* [WidgetsBinding], which handles the widget tree.
  
并且类中只有一个`ensureInitialized()`方法用来初始化`WidgetsBinding`对象,接着去执行了`scheduleAttachRootWidget`、`scheduleWarmUpFrame`方法
在`ensureInitialized`方法中调用`WidgetsFlutterBinding`进行了初始化
``` dart
static WidgetsBinding ensureInitialized() {
  if (WidgetsBinding._instance == null)
    WidgetsFlutterBinding();
  return WidgetsBinding.instance;
}
```

### scheduleAttachRootWidget

接着看 scheduleAttachRootWidget这个方法中执行了
``` dart
Timer.run(() {
  attachRootWidget(rootWidget);
});
``` 

在attachRootWidget方法中

``` dart
  void attachRootWidget(Widget rootWidget) {
    final bool isBootstrapFrame = renderViewElement == null;
    _readyToProduceFrames = true;
    _renderViewElement = RenderObjectToWidgetAdapter<RenderBox>(
      container: renderView,
      debugShortDescription: '[root]',
      child: rootWidget,
    ).attachToRenderTree(buildOwner!, renderViewElement as RenderObjectToWidgetElement<RenderBox>?);
    if (isBootstrapFrame) {
      SchedulerBinding.instance.ensureVisualUpdate();
    }
  }
``` 

注意看这里的`_renderViewElement`对象是由`RenderObjectToWidgetAdapter.attachToRenderTree()`返回的;
在初始化`RenderObjectToWidgetAdapter`对象时传入了`renderView` 和`rootWidget`作为参数,这里的`rootWidget`就是我们`runApp`中传入的参数;
那么这里的`renderView`是什么时候初始化的?我们在上面提到的`WidgetsFlutterBinding`混入了七个`xxxbinding`,这里需要了解mixin的执行顺序:
虽然首先执行的是`WidgetsBinding`的`initInstances`方法,但由于第一就执行了`super.initInstances()`,所以会先执行前一个`RenderBinding`的`initInstances`,然后不断super,所以最终`initInstances`实际的逻辑执行顺序,可以看成是从前面的Binding往后面的Binding,所以在`WidgetsBinding`的`attachRootWidget`方法内`renderView`已经被初始化了.

### RenderObjectToWidgetAdapter
继承自`RenderObjectWidget`,它有两个关键的抽象方法
``` dart
RenderObjectElement createElement();
RenderObject createRenderObject(BuildContext context);
```
看下是怎么覆写的
``` dart
@override
RenderObjectToWidgetElement<T> createElement() => RenderObjectToWidgetElement<T>(this);

@override
RenderObjectWithChildMixin<T> createRenderObject(BuildContext context) => container;
``` 
我们接着看attachToRenderTree

``` dart 
  RenderObjectToWidgetElement<T> attachToRenderTree(BuildOwner owner, [ RenderObjectToWidgetElement<T>? element ]) {
    if (element == null) {
      owner.lockState(() {
        element = createElement();
        assert(element != null);
        element!.assignOwner(owner);
      });
      owner.buildScope(element!, () {
        element!.mount(null, null);
      });
    } else {
      element._newWidget = this;
      element.markNeedsBuild();
    }
    return element!;
  }
```
这里传入了`BuildOwner`的实例`owner`和根元素对象.首先执行了`owner.lockState`,这个方法只是进行了一些断言来保证执行`callback`期间状态的锁定,这里`callback`就是4~6行代码;
在这个`callback`中执行了`createElement()`,用于创建元素,创建出来的元素也就是树的根节点;这里注意一下`createElement`是`RenderObjectToWidgetAdapter`实例的方法,看下上面的方法中传入的`this`也就是`RenderObjectToWidgetAdapter`对象本身;那么在创建Element时为啥要传入Widget对象？跟踪到最父级的Element发现是为了给_widget赋值.*也就是说Element持有了Widget对象,并且该元素由该组件创建*

## 挂载

接下来是`owner.buildScope`,这里传入了根元素和回调函数,同样的是进行了一些断言后回调了`callback`,在`callback`中执行了元素的挂载,注意这里传入的两个参数都是`null`.
``` dart
  @override
  void mount(Element? parent, Object? newSlot) {
    assert(parent == null);
    super.mount(parent, newSlot);
    _rebuild();
    assert(_child != null);
  }
``` 
`mount`方法是`RenderObjectToWidgetElement`类覆写的`Element`中定义的方法,这里执行了父类的`mount`方法和`_rebuild`方法;
先看mount的调用路径
`RootRenderObjectElement-->RenderObjectElement-->Element`
``` dart
  void mount(Element? parent, Object? newSlot) {
    assert(_lifecycleState == _ElementLifecycle.initial);
    assert(widget != null);
    assert(_parent == null);
    assert(parent == null || parent._lifecycleState == _ElementLifecycle.active);
    assert(slot == null);
    _parent = parent;
    _slot = newSlot;
    _lifecycleState = _ElementLifecycle.active;
    _depth = _parent != null ? _parent!.depth + 1 : 1;
    if (parent != null) {
      // Only assign ownership if the parent is non-null. If parent is null
      // (the root node), the owner should have already been assigned.
      // See RootRenderObjectElement.assignOwner().
      _owner = parent.owner;
    }
    assert(owner != null);
    final Key? key = widget.key;
    if (key is GlobalKey) {
      owner!._registerGlobalKey(key, this);
    }
    _updateInheritance();
    attachNotificationTree();
  }
```
这里维护了一些成员信息,并将树的深度_depth加1,到这里也就以为着根元素节点挂载完成
当`Element#mount`执行完成后,回到`RenderObjectToWidgetElement#mount`
``` dart 
  @override
  void mount(Element? parent, Object? newSlot) {
    super.mount(parent, newSlot);
    assert(() {
      _debugDoingBuild = true;
      return true;
    }());
    _renderObject = (widget as RenderObjectWidget).createRenderObject(this);
    assert(!_renderObject!.debugDisposed!);
    assert(() {
      _debugDoingBuild = false;
      return true;
    }());
    assert(() {
      _debugUpdateRenderObjectOwner();
      return true;
    }());
    assert(_slot == newSlot);
    attachRenderObject(newSlot);
    _dirty = false;
  }
```
这里面执行了`widget`的`createRenderObject(this)`方法来创建`_renderObject`;注意一下,这里的`widget`其实就是根组件.也就是`RenderObjectToWidgetAdapter`的实例对象,调用其`createRenderObject`方法返回的是其实例中的`container`对象,也就是说`Element`中的`_renderObject`是在`mount`方法中通过`widget.createRenderObject`方法创建的
``` dart 
@override
RenderObjectWithChildMixin<T> createRenderObject(BuildContext context) => container;
``` 
这里的`container`对象也就是前面提到的`attachRootWidget`中传入的`renderView`对象.
对于根节点的三棵树来讲,已经完成了创建过程,单着并不代表所有的节点都是这中情况.一般情况下,组件不会持有渲染对象,只不过根组件比较特殊,需要有一个开始渲染的节点,`createRenderObject`方法返回的RenderView也有特殊性

总结一下

* `RenderObjectToWidgetAdapter`通过构造方法持有`RenderView`对象
* `RenderObjectToWidgetAdapter`通过`createElement`方法创建`RenderObjectToWidgetElement`对象
* `RenderObjectToWidgetElement`通过`mount`方法持有`RenderView`
* `RenderObjectToWidgetElement`通过构造方法(Element)持有`RenderObjectToWidgetAdapter`

### 根渲染对象的关联
挂载完了我们接着看`RenderObjectElement#mount`方法中调用的`attachRenderObject(newSlot)`
``` dart 
  @override
  void attachRenderObject(Object? newSlot) {
    assert(_ancestorRenderObjectElement == null);
    _slot = newSlot;
    _ancestorRenderObjectElement = _findAncestorRenderObjectElement();
    _ancestorRenderObjectElement?.insertRenderObjectChild(renderObject, newSlot);
    final ParentDataElement<ParentData>? parentDataElement = _findAncestorParentDataElement();
    if (parentDataElement != null)
      _updateParentData(parentDataElement.widget as ParentDataWidget<ParentData>);
  }
```
先调用`_findAncestorRenderObjectElement从`元素树中向上查找第一个`RenderObjectElement`类型的元素节点作为先祖节点,然后调用其`insertRenderObjectChild`方法将自身持有的`renderObject`插入的渲染树中;
然后调用`_findAncestorParentDataElement`方法从元素树中向上查找第一个`ParentDataElement<ParentData>`类型的节点,如果非空,则执行`_updateParentData`方法;由于当前是根节点,这两个查找的方法返回的都是空
### 节点挂载
当父类的mount方法执行完毕后,回过头来看`RenderObjectToWidgetElement#mount`方法中调用的`_rebuild()`方法.
``` dart 
  void _rebuild() {
    try {
      _child = updateChild(_child, (widget as RenderObjectToWidgetAdapter<T>).child, _rootChildSlot);
    } catch (exception, stack) {
      final FlutterErrorDetails details = FlutterErrorDetails(
        exception: exception,
        stack: stack,
        library: 'widgets library',
        context: ErrorDescription('attaching to the render tree'),
      );
      FlutterError.reportError(details);
      final Widget error = ErrorWidget.builder(details);
      _child = updateChild(null, error, _rootChildSlot);
    }
  }
```
调用了`updateChild`方法,这里面有三个参数.第一个参数`_child`现在为`null`,最后一个`_rootChildSlot`是一个`object`,注意一下第二个参数`widget.child`:这里的widget是root也就是`RenderObjectToWidgetAdapter`对象的实例,它的`child`也就是是我们在`runApp`中传入的`widget`对象,也就是我们在前面`attachRootWidget`方法中创建`RenderObjectToWidgetAdapter`时传入的`child`参数.

接着看updateChild方法,我们在注释中找到了行为说明
| | newWidget == null|newWidget != null |
| --- | --- | --- |
|child == null|Returns null|Returns new [Element]|
|child != null|	Old child is removed, returns null|Old child updated if possible, returns child or new [Element]|
``` dart 
  Element? updateChild(Element? child, Widget? newWidget, Object? newSlot) {
    if (newWidget == null) {...}
    final Element newChild;
    if (child != null) {...} else {
      // The [debugProfileBuildsEnabled] code for this branch is inside
      // [inflateWidget], since some [Element]s call [inflateWidget] directly
      // instead of going through [updateChild].
      newChild = inflateWidget(newWidget, newSlot);
    }
    assert(...);
    return newChild;
  }
``` 
为了节省篇幅,这里删除了没有执行的代码;因为这里的child为空,所以会走inflateWidget(newWidget, newSlot)方法
``` dart 
  Element inflateWidget(Widget newWidget, Object? newSlot) {
    try {
      final Key? key = newWidget.key;
      if (key is GlobalKey) {
        final Element? newChild = _retakeInactiveElement(key, newWidget);
        if (newChild != null) {
          newChild._activateWithParent(this, newSlot);
          final Element? updatedChild = updateChild(newChild, newWidget, newSlot);
          return updatedChild!;
        }
      }
      final Element newChild = newWidget.createElement();
      newChild.mount(this, newSlot);
      return newChild;
    } finally {
      if (isTimelineTracked)
        Timeline.finishSync();
    }
  }
``` 
这里检查了组件是否有key并且key是不是GlobalKey.这里先放一下
后面调用`newWidget.createElement()`创建了`element`,并且调用其`mount`进行挂载.
然后就开始了树的遍历进行挂载,根据我们在`runApp`中传入的组件不同,调用不同对象的方法,