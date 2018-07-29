---
title: View事件分发
date: 2018-07-29 11:28:20
tags: [Android]
---

抄的《Android开发艺术探索》3.4.1 和 3.4.2

###### MotionEvent
在手指接触屏幕后所产生的一系列事件中，典型的事件类型有如下几种：
* ACTION_DOWN——手指刚接触屏幕；
* ACTION_MOVE——手指在屏幕上移动；
* ACTION_UP——手机从屏幕上松开的一瞬间
还有其他的事件可以参考`MotionEvent.java`类

###### TouchSlop
TouchSlop是系统所能识别出的被认为是滑动的最小距离，换句话说，当手指在屏幕上滑动时，如果两次滑动之间的距离小于这个常量，那么系统就不认为你是在进行滑动操作，这是一个常量，和设备有关，在不同设备上这个值可能是不同的，通过如下方式即可获取这个常量：`ViewConfiguration. get(getContext()).getScaledTouchSlop()`。当我们在处理滑动时，可以利用这个常量来做一些过滤，比如当两次滑动事件的滑动距离小于这个值，我们就可以认为未达到滑动距离的临界值，因此就可以认为它们不是滑动，这样做可以有更好的用户体验可以在源码中找到这个常量的定义，在`frameworks/base/core/res/res/values/config.xml`文件中。

<!--more-->
所谓点击事件的事件分发，其实就是对MotionEvent事件的分发过程，即当一个MotionEvent产生了以后，系统需要把这个事件传递给一个具体的View，而这个传递的过程就是分发过程。点击事件的分发过程由三个很重要的方法来共同完成：dispatchTouchEvent、onInterceptTouchEvent和onTouchEvent，下面我们先介绍一下这几个方法

###### public boolean dispatchTouchEvent(MotionEvent ev)
用来进行事件的分发。如果事件能够传递给当前View，那么此方法一定会被调用，返回结果受当前View的onTouchEvent和下级View的dispatchTouchEvent方法的影响，表示是否消耗当前事件。

###### public boolean onInterceptTouchEvent(MotionEvent event)
在上述方法内部调用，用来判断是否拦截某个事件，如果当前View拦截了某个事件，那么在同一个事件序列当中，此方法不会被再次调用，返回结果表示是否拦截当前事件。

###### public boolean onTouchEvent(MotionEvent event)
在dispatchTouchEvent方法中调用，用来处理点击事件，返回结果表示是否消耗当前事件，如果不消耗，则在同一个事件序列中，当前View无法再次接收到事件.

``` java
public boolean dispatchTouchEvent(MotionEvent ev) {
    boolean consume = false;
    if (onInterceptTouchEvent(ev)) {
        consume = onTouchEvent(ev);
    } else {
        consume = child.dispatchTouchEvent(ev);
    }
    return consume;
}
```
对于一个根`ViewGroup`来说，点击事件产生后，首先会传递给它，这时它的`dispatchTouchEvent`就会被调用，如果这个`ViewGroup`的`onInterceptTouchEvent`方法返回`true`就表示它要拦截当前事件，接着事件就会交给这个`ViewGroup`处理，即它的`onTouchEvent`方法就会被调用；如果这个`ViewGroup`的`onInterceptTouchEvent`方法返回`false`就表示它不拦截当前事件，这时当前事件就会继续传递给它的子元素，接着子元素的`dispatchTouchEvent`方法就会被调用，如此反复直到事件被最终处理。
当一个`View`需要处理事件时，如果它设置了`OnTouchListener`，那么`OnTouchListener`中的`onTouch`方法会被回调。这时事件如何处理还要看`onTouch`的返回值，如果返回`false`，则当前`View`的`onTouchEvent`方法会被调用；如果返回`true`，那么`onTouchEvent`方法将不会被调用。由此可见，给`View`设置的`OnTouchListener`，其优先级比`onTouchEvent`要高。在`onTouchEvent`方法中，如果当前设置的有`OnClickListener`，那么它的`onClick`方法会被调用。可以看出，平时我们常用的`OnClickListener`，其优先级最低，即处于事件传递的尾端。
当一个点击事件产生后，它的传递过程遵循如下顺序：`Activity -> Window -> View`，即事件总是先传递给`Activity`，`Activity`再传递给`Window`，最后`Window`再传递给顶级`View`。顶级`View`接收到事件后，就会按照事件分发机制去分发事件。考虑一种情况，如果一个`View`的`onTouchEvent`返回`false`，那么它的父容器的`onTouchEvent`将会被调用，依此类推。如果所有的元素都不处理这个事件，那么这个事件将会最终传递给`Activity`处理，即`Activity`的`onTouchEvent`方法会被调用。

###### 事件传递机制

* 同一个事件序列是指从手指接触屏幕的那一刻起，到手指离开屏幕的那一刻结束，在这个过程中所产生的一系列事件，这个事件序列以down事件开始，中间含有数量不定的move事件，最终以up事件结束。
* 正常情况下，一个事件序列只能被一个View拦截且消耗。因为一旦一个元素拦截了某此事件，那么同一个事件序列内的所有事件都会直接交给它处理，因此同一个事件序列中的事件不能分别由两个View同时处理，但是通过特殊手段可以做到，比如一个View将本该自己处理的事件通过onTouchEvent强行传递给其他View处理。
* 某个View一旦决定拦截，那么这一个事件序列都只能由它来处理（如果事件序列能够传递给它的话），并且它的onInterceptTouchEvent不会再被调用。这条也很好理解，就是说当一个View决定拦截一个事件后，那么系统会把同一个事件序列内的其他方法都直接交给它来处理，因此就不用再调用这个View的onInterceptTouchEvent去询问它是否要拦截了。
* 某个View一旦开始处理事件，如果它不消耗ACTION_DOWN事件（onTouchEvent返回了false），那么同一事件序列中的其他事件都不会再交给它来处理，并且事件将重新交由它的父元素去处理，即父元素的onTouchEvent会被调用。意思就是事件一旦交给一个View处理，那么它就必须消耗掉，否则同一事件序列中剩下的事件就不再交给它来处理了。
* 如果View不消耗除ACTION_DOWN以外的其他事件，那么这个点击事件会消失，此时父元素的onTouchEvent并不会被调用，并且当前View可以持续收到后续的事件，最终这些消失的点击事件会传递给Activity处理。
* ViewGroup默认不拦截任何事件。Android源码中ViewGroup的onInterceptTouch-Event方法默认返回false。
* View没有onInterceptTouchEvent方法，一旦有点击事件传递给它，那么它的onTouchEvent方法就会被调用。
* View的onTouchEvent默认都会消耗事件（返回true），除非它是不可点击的（clickable  和longClickable同时为false）。View的longClickable属性默认都为false，clickable属性要分情况，比如Button的clickable属性默认为true，而TextView的clickable属性默认为false。
* View的enable属性不影响onTouchEvent的默认返回值。哪怕一个View是disable状态的，只要它的clickable或者longClickable有一个为true，那么它的onTouchEvent就返回true。
* onClick会发生的前提是当前View是可点击的，并且它收到了down和up的事件。
* 事件传递过程是由外向内的，即事件总是先传递给父元素，然后再由父元素分发给子View，通过requestDisallowInterceptTouchEvent方法可以在子元素中干预父元素的事件分发过程，但是ACTION_DOWN事件除外。

###### 事件分发源码
** Activity对点击事件的分发过程 **
点击事件用`MotionEvent`来表示，当一个点击操作发生时，事件最先传递给当前`Activity`，由`Activity`的`dispatchTouchEvent`来进行事件派发，具体的工作是由`Activity`内部的`Window`来完成的。`Window`会将事件传递给`decor view`，`decor view`一般就是当前界面的底层容器（即`setContentView`所设置的`View`的父容器），通过`Activity.getWindow.getDecorView()`可以获得。我们先从`Activity`的`dispatchTouchEvent`开始分析。
``` java
 /**
     * Called to process touch screen events.  You can override this to
     * intercept all touch screen events before they are dispatched to the
     * window.  Be sure to call this implementation for touch screen events
     * that should be handled normally.
     *
     * @param ev The touch screen event.
     *
     * @return boolean Return true if this event was consumed.
     */
    public boolean dispatchTouchEvent(MotionEvent ev) {
        if (ev.getAction() == MotionEvent.ACTION_DOWN) {
            onUserInteraction();
        }
        if (getWindow().superDispatchTouchEvent(ev)) {
            return true;
        }
        return onTouchEvent(ev);
    }
```
在window类的注释中

```
/**
 * Abstract base class for a top-level window look and behavior policy.  An
 * instance of this class should be used as the top-level view added to the
 * window manager. It provides standard UI policies such as a background, title
 * area, default key processing, etc.
 *
 * <p>The only existing implementation of this abstract class is
 * android.view.PhoneWindow, which you should instantiate when needing a
 * Window.
 */
```

首先事件开始交给`Activity`所附属的`Window`进行分发，如果返回`true`，整个事件循环就结束了，返回`false`意味着事件没人处理，所有`View`的`onTouchEvent`都返回了`false`，那么`Activity`的`onTouchEvent`就会被调用。
其中`Window`是个抽象类，而其中的`superDispatchTouchEvent`方法也是个抽象方法。在`PhoneWindows`中
``` java
 @Override
    public boolean superDispatchTouchEvent(MotionEvent event) {
        return mDecor.superDispatchTouchEvent(event);
    }
```
调用了DecorView的superDispatchTouchEvent方法，我们可以看一下DecorView：
``` java
public class DecorView extends FrameLayout implements RootViewSurfaceTaker, WindowCallbacks {
public boolean superDispatchTouchEvent(MotionEvent event) {
        return super.dispatchTouchEvent(event);
    }
}
```
目前事件传递到了`DecorView`这里，由于`DecorView`继承自`FrameLayout`且是父`View`，所以最终事件会传递给`View`。从这里开始，事件已经传递到顶级`View`了，即在`Activity`中通过`setContentView`所设置的`View`，另外顶级`View`也叫根`View`，顶级View一般来说都是`ViewGroup`。

** 顶级View对点击事件的分发过程 **
点击事件达到顶级`View`（一般是一个ViewGroup）以后，会调用`ViewGroup的dispatchTouchEvent`方法，然后的逻辑是这样的：如果顶级`ViewGroup`拦截事件即`onInterceptTouchEvent`返回`true`，则事件由`ViewGroup`处理，这时如果`ViewGroup`的`mOnTouchListener`被设置，则`onTouch`会被调用，否则`onTouchEvent`会被调用。也就是说，如果都提供的话，`onTouch`会屏蔽掉`onTouchEven`t。在`onTouchEvent`中，如果设置了`mOnClickListener`，则`onClick`会被调用。如果顶级`ViewGroup`不拦截事件，则事件会传递给它所在的点击事件链上的子`View`，这时子`View`的`dispatchTouchEvent`会被调用。到此为止，事件已经从顶级View传递给了下一层`View`，接下来的传递过程和顶级`View`是一致的，如此循环，完成整个事件的分发。具体代码可以看一下`ViewGroup.dispatchTouchEvent()`方法。

** View对点击事件的处理 **
View对点击事件的处理过程稍微简单一些，这里的View不包含ViewGroup。
View对点击事件的处理过程就比较简单了，因为View（这里不包含ViewGroup）是一个单独的元素，它没有子元素因此无法向下传递事件，所以它只能自己处理事件。从上面的源码可以看出View对点击事件的处理过程，首先会判断有没有设置OnTouchListener，如果OnTouchListener中的onTouch方法返回true，那么onTouchEvent就不会被调用，可见OnTouchListener的优先级高于onTouchEvent，这样做的好处是方便在外界处理点击事件。接着再分析onTouchEvent的实现。先看当View处于不可用状态下点击事件的处理过程，如下所示。很显然，不可用状态下的View照样会消耗点击事件，尽管它看起来不可用。
接着再分析onTouchEvent的实现。先看当View处于不可用状态下点击事件的处理过程，如下所示。很显然，不可用状态下的View照样会消耗点击事件，尽管它看起来不可用。
``` java
 if ((viewFlags & ENABLED_MASK) == DISABLED) {
            if (action == MotionEvent.ACTION_UP && (mPrivateFlags & PFLAG_PRESSED) != 0) {
                setPressed(false);
            }
            mPrivateFlags3 &= ~PFLAG3_FINGER_DOWN;
            // A disabled view that is clickable still consumes the touch
            // events, it just doesn't respond to them.
            return clickable;
        }
```
接下来如果View设置有代理，还会执行代理的onTouchEvent方法，
``` java
if (mTouchDelegate != null) {
            if (mTouchDelegate.onTouchEvent(event)) {
                return true;
            }
        }
```

接下来就是对事件序列的处理
``` java
if (clickable || (viewFlags & TOOLTIP) == TOOLTIP) {
            switch (action) {
                case MotionEvent.ACTION_UP:break;
                case MotionEvent.ACTION_DOWN:break;
                case MotionEvent.ACTION_CANCEL:break;
                case MotionEvent.ACTION_MOVE:break;
            }
}
```
从上面的代码来看，只要View的`CLICKABLE`和`LONG_CLICKABLE`有一个为true，那么它就会消耗这个事件，即`onTouchEvent`方法返回true，不管它是不是`DISABLE`状态，然后就是当`ACTION_UP`事件发生时，会触发`performClick`方法，如果View设置了`OnClickListener`，那么`performClick`方法内部会调用它的`onClick`方法。`View`的`LONG_CLICKABLE`属性默认为`false`，而`CLICKABLE`属性是否为`false`和具体的View有关，确切来说是可点击的View其`CLICKABLE为true`，不可点击的View其`CLICKABLE`为`false`，比如`Button`是可点击的，`TextView`是不可点击的。通过`setClickable`和`setLongClickable`可以分别改变`View`的`CLICKABLE`和`LONG_CLICKABLE`属性。另外，`setOnClickListener`会自动将`View`的`CLICKABLE`设为`true`，s`etOnLongClickListener`则会自动将`View`的`LONG_CLICKABLE`设为`true`。

###### 处理滑动冲突
了解了点击事件的处理过程，就可以比较好的处理滑动冲突了

** 父容器拦截处理 **
击事情都先经过父容器的拦截处理，如果父容器需要此事件就拦截，如果不需要此事件就不拦截。外部拦截法需要重写父容器的onInterceptTouchEvent方法，在内部做相应的拦截即可。
``` java
public boolean onInterceptTouchEvent(MotionEvent event) {
    boolean intercepted = false;
    int x = (int) event.getX();
    int y = (int) event.getY();
    switch (event.getAction()) {
        case MotionEvent.ACTION_DOWN: {
            intercepted = false;
            break;
        }
        case MotionEvent.ACTION_MOVE: {
            if (父容器需要当前点击事件) {
                intercepted = true;
            } else {
                intercepted = false;
            }
            break;
        }
        case MotionEvent.ACTION_UP: {
            intercepted = false;
            break;
        }
        default:
            break;
    }
    mLastXIntercept = x;
    mLastYIntercept = y;
    return intercepted;
}
```
针对不同的滑动冲突，只需要修改父容器需要当前点击事件这个条件即可，其他均不需做修改并且也不能修改。这里对上述代码再描述一下，在`onInterceptTouchEvent`方法中，首先是`ACTION_DOWN`这个事件，父容器必须返回`false`，即不拦截`ACTION_DOWN`事件，这是因为一旦父容器拦截了`ACTION_DOWN`，那么后续的`ACTION_MOVE`和`ACTION_UP`事件都会直接交由父容器处理，这个时候事件没法再传递给子元素了；其次是`ACTION_MOVE`事件，这个事件可以根据需要来决定是否拦截，如果父容器需要拦截就返回`true`，否则返回`false`；最后是`ACTION_UP`事件，这里必须要返回false，因为`ACTION_UP`事件本身没有太多意义。假设事件交由子元素处理，如果父容器在`ACTION_UP`时返回了`true`，就会导致子元素无法接收到`ACTION_UP`事件，这个时候子元素中的`onClick`事件就无法触发，但是父容器比较特殊，一旦它开始拦截任何一个事件，那么后续的事件都会交给它来处理，而`ACTION_UP`作为最后一个事件也必定可以传递给父容器，即便父容器的`onInterceptTouchEvent`方法在`ACTION_UP`时返回了`false`。

** 子元素拦截事件 **
父容器不拦截任何事件，所有的事件都传递给子元素，如果子元素需要此事件就直接消耗掉，否则就交由父容器进行处理，这种方法和Android中的事件分发机制不一致，需要配合`requestDisallowInterceptTouchEvent()`方法才能正常工作，使用起来较外部拦截法稍显复杂。
``` java
public boolean dispatchTouchEvent(MotionEvent event) {
    int x = (int) event.getX();
    int y = (int) event.getY();
    switch (event.getAction()) {
        case MotionEvent.ACTION_DOWN: {
            parent.requestDisallowInterceptTouchEvent(true);
            break;
        }
        case MotionEvent.ACTION_MOVE: {
            int deltaX = x -mLastX;
            int deltaY = y -mLastY;
            if (父容器需要此类点击事件)) {
                parent.requestDisallowInterceptTouchEvent(false);
            }
            break;
        }
        case MotionEvent.ACTION_UP: {
            break;
        }
        default:
            break;
        }
    mLastX = x;
    mLastY = y;
    return super.dispatchTouchEvent(event);
}
```
除了子元素需要做处理以外，父元素也要默认拦截除了`ACTION_DOWN`以外的其他事件，这样当子元素调用`parent.requestDisallowInterceptTouchEvent(false)`方法时，父元素才能继续拦截所需的事件。因为`ACTION_DOWN`事件并不受`FLAG_DISALLOW_INTERCEPT`这个标记位的控制，所以一旦父容器拦截`ACTION_DOWN`事件，那么所有的事件都无法传递到子元素中去，这样内部拦截就无法起作用了。

----
以上