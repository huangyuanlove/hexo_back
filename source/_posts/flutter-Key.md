---
title: flutter_Key
tags: [Flutter]
date: 2023-01-23 14:31:47
keywords: [Flutter Key,GlobalKey,LocalKey,UniqueKey,ObjectKey,ValueKey]
---

官方视频在这里，有条件的可以看下 
https://www.youtube.com/watch?v=kn0EOS-ZiIc
也可以看下这个对diff算法的详解
https://juejin.cn/post/6935422635194974244
主要代码如下
<!--more-->
一个用于获取颜色的RandomColor
``` dart 
class RandomColor {
  static final Random _random = Random();

  static Color getColor() {
    return Color.fromRGBO(
      _random.nextInt(256),
      _random.nextInt(256),
      _random.nextInt(256),
      1,
    );
  }
}
``` 
一个继承自StatefulWidget的widget，使用State保存了颜色信息
``` dart
class RandomColorBoxStateful extends StatefulWidget {
  RandomColorBoxStateful({Key? key}) : super(key: key);

  @override
  State<RandomColorBoxStateful> createState() => RandomColorBoxState();
}

class RandomColorBoxState extends State<RandomColorBoxStateful> {

  late Color myColor;
  @override
  void initState() {
    super.initState();
    myColor = RandomColor.getColor();
  }

  @override
  Widget build(BuildContext context) {
    return Text("$myColor",style: TextStyle(color: myColor),);
  }
}
```
一个继承自StatelessWidget的widget，内容差不多
``` dart
class RandomColorBoxStateless extends StatelessWidget {
   RandomColorBoxStateless({Key? key}) : super(key: key);
  Color myColor = RandomColor.getColor();

  @override
  Widget build(BuildContext context) {
      return Text("$myColor",style: TextStyle(color: myColor),););
  }
}
```
一个用来显示界面的SwapColorBox
``` dart
class SwapColorBox extends StatefulWidget {
  @override
  State<StatefulWidget> createState() => SwapColorBoxState();
}

class SwapColorBoxState extends State<SwapColorBox> {
  List<Widget> tiles = [ RandomColorBoxStateful(), RandomColorBoxStateful() ];

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: SafeArea(child: Column(children: tiles)),
      floatingActionButton: FloatingActionButton(
        child: Icon(Icons.sentiment_very_satisfied),
        onPressed: swapTiles,
      ),
    );
  }

  void swapTiles() {
    setState(() {
      List<Widget> tmp = [tiles[1],tiles[0]];
      tiles = tmp;
    });
  }
}
``` 
这时点击floatingActionButton会发现页面没有变化。
### 如何修改
好几种办法
* 将SwapColorBoxState中的tiles改为 List<Widget> tiles = [ RandomColorBoxStateless(), RandomColorBoxStateless() ]
* 将SwapColorBoxState中的tiles中RandomColorBoxStateful加上UniqueKey : List<Widget> tiles = [ RandomColorBoxStateful(key: UniqueKey(), ), RandomColorBoxStateful(key: UniqueKey(), ) ];
* 将RandomColorBoxStateful中的myColor放在RandomColorBoxStateful中而不是RandomColorBoxState中

### 为什么
元素树没有交换，虽然我们交换了Widget，但是其Element并没有交换，而颜色状态又是由State维护，所以在执行build的时候颜色并没有变化。
先看下更新的代码
``` dart 
  List<Element> updateChildren(List<Element> oldChildren, List<Widget> newWidgets, { Set<Element>? forgottenChildren, List<Object?>? slots }) {


    Element? replaceWithNullIfForgotten(Element child) {
      return forgottenChildren != null && forgottenChildren.contains(child) ? null : child;
    }

    Object? slotFor(int newChildIndex, Element? previousChild) {
      return slots != null
        ? slots[newChildIndex]
        : IndexedSlot<Element?>(newChildIndex, previousChild);
    }


    int newChildrenTop = 0;
    int oldChildrenTop = 0;
    int newChildrenBottom = newWidgets.length - 1;
    int oldChildrenBottom = oldChildren.length - 1;

    final List<Element> newChildren = oldChildren.length == newWidgets.length ?
        oldChildren : List<Element>.filled(newWidgets.length, _NullElement.instance);

    Element? previousChild;

    // Update the top of the list.
    while ((oldChildrenTop <= oldChildrenBottom) && (newChildrenTop <= newChildrenBottom)) {
      final Element? oldChild = replaceWithNullIfForgotten(oldChildren[oldChildrenTop]);
      final Widget newWidget = newWidgets[newChildrenTop];
     
      if (oldChild == null || !Widget.canUpdate(oldChild.widget, newWidget))
        break;
      final Element newChild = updateChild(oldChild, newWidget, slotFor(newChildrenTop, previousChild))!;
      
      newChildren[newChildrenTop] = newChild;
      previousChild = newChild;
      newChildrenTop += 1;
      oldChildrenTop += 1;
    }

    // Scan the bottom of the list.
    while ((oldChildrenTop <= oldChildrenBottom) && (newChildrenTop <= newChildrenBottom)) {
      final Element? oldChild = replaceWithNullIfForgotten(oldChildren[oldChildrenBottom]);
      final Widget newWidget = newWidgets[newChildrenBottom];
      
      if (oldChild == null || !Widget.canUpdate(oldChild.widget, newWidget))
        break;
      oldChildrenBottom -= 1;
      newChildrenBottom -= 1;
    }

    // Scan the old children in the middle of the list.
    final bool haveOldChildren = oldChildrenTop <= oldChildrenBottom;
    Map<Key, Element>? oldKeyedChildren;
    if (haveOldChildren) {
      oldKeyedChildren = <Key, Element>{};
      while (oldChildrenTop <= oldChildrenBottom) {
        final Element? oldChild = replaceWithNullIfForgotten(oldChildren[oldChildrenTop]);
       
        if (oldChild != null) {
          if (oldChild.widget.key != null)
            oldKeyedChildren[oldChild.widget.key!] = oldChild;
          else
            deactivateChild(oldChild);
        }
        oldChildrenTop += 1;
      }
    }

    // Update the middle of the list.
    while (newChildrenTop <= newChildrenBottom) {
      Element? oldChild;
      final Widget newWidget = newWidgets[newChildrenTop];
      if (haveOldChildren) {
        final Key? key = newWidget.key;
        if (key != null) {
          oldChild = oldKeyedChildren![key];
          if (oldChild != null) {
            if (Widget.canUpdate(oldChild.widget, newWidget)) {
              // we found a match!
              // remove it from oldKeyedChildren so we don't unsync it later
              oldKeyedChildren.remove(key);
            } else {
              // Not a match, let's pretend we didn't see it for now.
              oldChild = null;
            }
          }
        }
      }
     
      final Element newChild = updateChild(oldChild, newWidget, slotFor(newChildrenTop, previousChild))!;
     
      newChildren[newChildrenTop] = newChild;
      previousChild = newChild;
      newChildrenTop += 1;
    }

    // We've scanned the whole list.
    
    newChildrenBottom = newWidgets.length - 1;
    oldChildrenBottom = oldChildren.length - 1;

    // Update the bottom of the list.
    while ((oldChildrenTop <= oldChildrenBottom) && (newChildrenTop <= newChildrenBottom)) {
      final Element oldChild = oldChildren[oldChildrenTop];
     
      final Widget newWidget = newWidgets[newChildrenTop];
     
      final Element newChild = updateChild(oldChild, newWidget, slotFor(newChildrenTop, previousChild))!;
     
      newChildren[newChildrenTop] = newChild;
      previousChild = newChild;
      newChildrenTop += 1;
      oldChildrenTop += 1;
    }

    // Clean up any of the remaining middle nodes from the old list.
    if (haveOldChildren && oldKeyedChildren!.isNotEmpty) {
      for (final Element oldChild in oldKeyedChildren.values) {
        if (forgottenChildren == null || !forgottenChildren.contains(oldChild))
          deactivateChild(oldChild);
      }
    }
   
    return newChildren;
  }
```
前置条件：
#### Widget.canUpdate()
``` dart
  static bool canUpdate(Widget oldWidget, Widget newWidget) {
    return oldWidget.runtimeType == newWidget.runtimeType
        && oldWidget.key == newWidget.key;
  }
```
比较两个Widget的runtimeType和key是否相同
####  Element.update()
``` dart
void update(covariant Widget newWidget) {
   _widget = newWidget;
}
```
只是简单的替换所持有Widget，并没有更新自己的其他属性

### 更新算法

framwork中将节点列表分成了三部分：顶部、中间部分、底部，当发生更新时，尽最大可能的复用Element，无法复用的才会去创建新的Element

1. 首先自顶向下的进行diff并更新子节点，也就是第一个while循环，是否能复用就是调用的canUpdate
2. 然后自底向上的进行diff(这里没有更新子节点)，也就是第二个while循环，依然是用canUpdate看判断是否可以复用
3. 然后在这两个中间的部分寻找可以复用的Element，并进行存储
4. 这时候就已经扫描完整棵树了，接下来更新中间部分
5. 最后更新底部
  
为什么在自底向上的进行diff时候没有更新：因为这时候拿不到Slot信息
回到我们上面提到的例子中点击按钮时会触发`Column`的更新，也就是`MultiChildRenderObjectElement`的更新，就会触发上面的`updateChildren()`方法
所以在自顶向下的更新中，`canUpdate()`返回的是`true`(当我们设置了Key之后，这里会返回false，不进行复用)，也就是可以复用`element`，接着执行了`updateChild(Element? child, Widget? newWidget, Object? newSlot)`
这里的`child`是*旧element*，`newWidget`也就是要显示的*widget*，两者并不相等，所以就执行了`child.update(newWidget);`只是简单的对所持有的`widget`进行了赋值。我们知道`StatefullWidget`中`State`和`StatefulElement`互相持有，并且两者都持有`StatefulWidget`。所以`State`并没有被更新，所持有的颜色值还是交换之前的颜色值，所以点击交换按钮后，`Widget`虽然交换了位置，但是`Element`并没有更新。
有点像是A机器生产A物品，B机器生产B物品；原来操作机器A的去操作机器B，原来操作机器B的去操作机器A，虽然换了操作员，但生产A的机器还是生产A，生产B的机器还是生产B。

### key
这里的key就两个分支`LocalKey` 和`GlobalKey` 。我们知道*key*的作用就是为`Widget`确认唯一的身份，可以在多子组件更新中被识别，这就是`LocalKey`的作用。所以`LocalKey`保证的是 **相同父级**组件的身份唯一性。而 `GlobalKey` 是整个应用中，组件的身份唯一。

`LocalKey`下面有`UniqueKey`、`ValueKey<T>`、`ObjectKey`,区别也很简单，戳进去看下源码就好了

#### Globalkey
对于`GlobalKey`来讲，只要获取到了`Element`，就能获取到`Widget`对象。只要`Element`是`StatefulElement`，就能获取到`State`.
那么如何获取到Element呢？
``` dart 
Element? get _currentElement => WidgetsBinding.instance.buildOwner!._globalKeyRegistry[this];

///BuildOwner
final Map<GlobalKey, Element> _globalKeyRegistry = <GlobalKey, Element>{};
void _registerGlobalKey(GlobalKey key, Element element) {
  _globalKeyRegistry[key] = element;
}
```
那么是在什么时候调用_registerGlobalKey注册的呢？前面提到的mount方法中
``` dart 
    if (key is GlobalKey) {
      owner!._registerGlobalKey(key, this);
    }
    ```
可以看到 就是在这里注册的。
并且会在unmount中进行反注册
``` dart
    final Key? key = _widget?.key;
    if (key is GlobalKey) {
      owner!._unregisterGlobalKey(key, this);
    }
```
源码中也对`GlobalKey`的使用场景做出了介绍，当你真的需要获取某个`BuildContext`或`State`时，用`GlobalKey`是完全没有问题的。