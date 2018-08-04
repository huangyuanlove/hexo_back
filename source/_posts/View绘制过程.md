---
title: View绘制过程
date: 2018-07-29 22:35:46
tags: [Android]
keywords: View绘制过程
---
抄的《Android开发艺术探索》第四章
`ViewRoot`对应于`ViewRootImpl`类，它是连接`WindowManager`和`DecorView`的纽带，`View`的三大流程均是通过`ViewRoot`来完成的。在`ActivityThread`中，当`Activity`对象被创建完毕后，会将`DecorView`添加到`Window`中，同时会创建`ViewRootImpl`对象，并将`ViewRootImpl`对象和`DecorView`建立关联，`View`的绘制流程是从`ViewRoot`的`performTraversals`方法开始的，它经过`measure`、`layout`和`draw`三个过程才能最终将一个`View`绘制出来，其中`measure`用来测量View的宽和高，`layout`用来确定View在父容器中的放置位置，而`draw`则负责将View绘制在屏幕上。

<!--more-->
`performTraversals`会依次调用`performMeasure`、p`erformLayout`和`performDraw`三个方法，这三个方法分别完成顶级`View`的`measure`、`layout`和`draw`这三大流程，其中在`performMeasure`中会调用`measure`方法，在`measure`方法中又会调用`onMeasure`方法，在`onMeasure`方法中则会对所有的子元素进行`measure`过程，这个时候`measure`流程就从父容器传递到子元素中了，这样就完成了一次`measure`过程。接着子元素会重复父容器的`measure`过程，如此反复就完成了整个`View`树的遍历。同理，`performLayout`和`performDraw`的传递流程和`performMeasure`是类似的，唯一不同的是，`performDraw`的传递过程是在`draw`方法中通过`dispatchDraw`来实现的，不过这并没有本质区别。

`measure`过程决定了`View`的宽/高，`Measure`完成以后，可以通过`getMeasuredWidth`和`getMeasuredHeight`方法来获取到`View`测量后的宽/高，在几乎所有的情况下它都等同于`View`最终的宽/高，但是特殊情况除外，这点在本章后面会进行说明。`Layout`过程决定了`View`的四个顶点的坐标和实际的View的宽/高，完成以后，可以通过`getTop`、`getBottom`、`getLeft`和`getRight`来拿到`View`的四个顶点的位置，并可以通过`getWidth`和`getHeight`方法来拿到`View`的最终宽/高。`Draw`过程则决定了`View`的显示，只有`draw`方法完成以后`View`的内容才能呈现在屏幕上。

`DecorView`作为顶级`View`，一般情况下它内部会包含一个竖直方向的`LinearLayout`，在这个`LinearLayout`里面有上下两个部分（具体情况和Android版本及主题有关），上面是标题栏，下面是内容栏。在`Activity`中我们通过`setContentView`所设置的布局文件其实就是被加到内容栏之中的，而内容栏的id是`content`，因此可以理解为Activity指定布局的方法不叫setview而叫`setContentView`，因为我们的布局的确加到了`id`为`content`的`FrameLayout`中。如何得到`content`呢？可以这样：`ViewGroup  content=  findViewById(R.android.id.content)`。如何得到我们设置的`View`呢？可以这样：`content.getChildAt(0)`。同时，通过源码我们可以知道，`DecorView`其实是一个`FrameLayout`，`View`层的事件都先经过`DecorView`，然后才传递给我们的View。

#### MeasureSpec
`MeasureSpec`在很大程度上决定了一个View的尺寸规格，之所以说是很大程度上是因为这个过程还受父容器的影响，因为父容器影响View的`MeasureSpec`的创建过程。在测量过程中，系统会将View的`LayoutParams`根据父容器所施加的规则转换成对应的`MeasureSpec`，然后再根据这个`measureSpec`来测量出`View`的宽/高。这里的宽/高是测量宽/高，不一定等于`View`的最终宽/高。
`MeasureSpec`代表一个32位int值，高2位代表`SpecMode`，低30位代表`SpecSize`，`SpecMode`是指测量模式，而`SpecSize`是指在某种测量模式下的规格大小。
``` java
private static final int MODE_SHIFT = 30;
private static final int MODE_MASK = 0x3 << MODE_SHIFT;
public static final int UNSPECIFIED = 0 << MODE_SHIFT;
public static final int EXACTLY = 1 << MODE_SHIFT;
public static final int AT_MOST = 2 << MODE_SHIFT;
public static int makeMeasureSpec(int size,int mode) {
    if (sUseBrokenMakeMeasureSpec) {
        return size + mode;
    } else {
        return (size & ~MODE_MASK) | (mode & MODE_MASK);
    }
}
public static int getMode(int measureSpec) {
    return (measureSpec & MODE_MASK);
}
public static int getSize(int measureSpec) {
    return (measureSpec & ~MODE_MASK);
}
```
`MeasureSpec`通过将`SpecMode`和`SpecSize`打包成一个int值来避免过多的对象内存分配，为了方便操作，其提供了打包和解包方法。`SpecMode`和`SpecSize`也是一个int值，一组`SpecMode`和`SpecSize`可以打包为一个`MeasureSpec`，而一个`MeasureSpec`可以通过解包的形式来得出其原始的`SpecMode`和`SpecSize`，需要注意的是这里提到的`MeasureSpec`是指`MeasureSpec`所代表的int值，而并非`MeasureSpec`本身。
SpecMode有三类，每一类都表示特殊的含义，如下所示。
** UNSPECIFIED **
父容器不对View有任何限制，要多大给多大，这种情况一般用于系统内部，表示一种测量的状态。
** EXACTLY **
父容器已经检测出View所需要的精确大小，这个时候View的最终大小就是SpecSize所指定的值。它对应于LayoutParams中的match_parent和具体的数值这两种模式。
** AT_MOST **
父容器指定了一个可用大小即SpecSize，View的大小不能大于这个值，具体是什么值要看不同View的具体实现。它对应于LayoutParams中的wrap_content。

###### MeasureSpec和LayoutParams
在View测量的时候，系统会将`LayoutParams`在父容器的约束下转换成对应的`MeasureSpec`，然后再根据这个`MeasureSpec`来确定View测量后的宽/高。需要注意的是，`MeasureSpec`不是唯一由`LayoutParams`决定的，`LayoutParams`需要和父容器一起才能决定`View`的`MeasureSpec`，从而进一步决定View的宽/高。另外，对于`顶级View`（即DecorView）和`普通View`来说，`MeasureSpec`的转换过程略有不同。对于`DecorView`，其`MeasureSpec`由窗口的尺寸和其自身的`LayoutParams`来共同确定；对于普通View，其`MeasureSpec`由父容器的`MeasureSpec`和自身的`LayoutParams`来共同决定，`MeasureSpec`一旦确定后，`onMeasure`中就可以确定`View`的测量宽/高。
对于`DecorView`来说，在`ViewRootImpl`中的`measureHierarchy`方法中有如下一段代码，它展示了`DecorView`的`MeasureSpec`的创建过程，其中`desiredWindowWidth`和`desiredWindowHeight`是屏幕的尺寸:
``` java
childWidthMeasureSpec = getRootMeasureSpec(desiredWindowWidth,lp.width);
childHeightMeasureSpec = getRootMeasureSpec(desiredWindowHeight,lp.height);
performMeasure(childWidthMeasureSpec,childHeightMeasureSpec);

private static int getRootMeasureSpec(int windowSize,int rootDimension) {
    int measureSpec;
    switch (rootDimension) {
        case ViewGroup.LayoutParams.MATCH_PARENT:
            // Window can't resize. Force root view to be windowSize.
            measureSpec = MeasureSpec.makeMeasureSpec(windowSize,MeasureSpec.EXACTLY);
            break;
        case ViewGroup.LayoutParams.WRAP_CONTENT:
            // Window can resize. Set max size for root view.
            measureSpec = MeasureSpec.makeMeasureSpec(windowSize,MeasureSpec.AT_MOST);
            break;
        default:
            // Window wants to be an exact size. Force root view to be that size.
            measureSpec = MeasureSpec.makeMeasureSpec(rootDimension,Measure-Spec.EXACTLY);
            break;
    }
    return measureSpec;
}
```
通过上述代码，`DecorView`的`MeasureSpec`的产生过程就很明确了，具体来说其遵守如下规则，根据它的`LayoutParams`中的宽/高的参数来划分。
* LayoutParams.MATCH_PARENT：精确模式，大小就是窗口的大小；
* LayoutParams.WRAP_CONTENT：最大模式，大小不定，但是不能超过窗口的大小；
* 固定大小（比如100dp）：精确模式，大小为LayoutParams中指定的大小。
对于普通`View`来说，这里是指我们布局中的`View`，`View`的`measure`过程由`ViewGroup`传递而来，先看一下`ViewGroup`的`measureChildWithMargins`方法：
``` java
protected void measureChildWithMargins(View child,int parentWidthMeasureSpec,int widthUsed,int parentHeightMeasureSpec,int heightUsed) {
    final MarginLayoutParams lp = (MarginLayoutParams) child.getLayout-Params();
    final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin+ widthUsed,lp.width);
    final int childHeightMeasureSpec = getChildMeasureSpec(parentHeight-MeasureSpec,mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin+ heightUsed,lp.height);
    child.measure(childWidthMeasureSpec,childHeightMeasureSpec);
}
```
上述方法会对子元素进行`measure`，在调用子元素的`measure`方法之前会先通过`getChildMeasureSpec`方法来得到子元素的`MeasureSpec`。从代码来看，很显然，子元素的`MeasureSpec`的创建与父容器的`MeasureSpec`和子元素本身的`LayoutParam`s有关，此外还和`View`的`margin`及`padding`有关，具体情况可以看一下`ViewGroup`的`getChildMeasureSpec`方法，如下所示:
``` java
public static int getChildMeasureSpec(int spec,int padding,int child-Dimension) {
    int specMode = MeasureSpec.getMode(spec);
    int specSize = MeasureSpec.getSize(spec);
    int size = Math.max(0,specSize -padding);
    int resultSize = 0;
    int resultMode = 0;
    switch (specMode) {
        // Parent has imposed an exact size on us
        case MeasureSpec.EXACTLY:
            if (childDimension => 0) {
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                // Child wants to be our size. So be it.
                resultSize = size;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                // Child wants to determine its own size. It can't be
                // bigger than us.
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            }
            break;
        // Parent has imposed a maximum size on us
        case MeasureSpec.AT_MOST:
            if (childDimension => 0) {
                // Child wants a specific size... so be it
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                // Child wants to be our size,but our size is not fixed.
                // Constrain child to not be bigger than us.
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                // Child wants to determine its own size. It can't be
                // bigger than us.
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            }
            break;
        // Parent asked to see how big we want to be
        case MeasureSpec.UNSPECIFIED:
            if (childDimension => 0) {
                // Child wants a specific size... let him have it
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                // Child wants to be our size... find out how big it should
                // be
                resultSize = 0;
                resultMode = MeasureSpec.UNSPECIFIED;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                // Child wants to determine its own size.... find out how
                // big it should be
                resultSize = 0;
                resultMode = MeasureSpec.UNSPECIFIED;
            }
            break;
        }
    return MeasureSpec.makeMeasureSpec(resultSize,resultMode);
}
```
它的主要作用是根据父容器的`MeasureSpec`同时结合View本身的`LayoutParams`来确定子元素的`MeasureSpec`，参数中的`padding`是指父容器中已占用的空间大小，因此子元素可用的大小为父容器的尺寸减去`padding`，具体代码如下所示:
``` java
    int specSize = MeasureSpec.getSize(spec);
    int size = Math.max(0,specSize -padding);
```
这里简单说一下，当View采用固定宽/高的时候，不管父容器的`MeasureSpec`是什么，`View`的`MeasureSpec`都是精确模式并且其大小遵循`Layoutparams`中的大小。当`View`的宽/高是`match_parent`时，如果父容器的模式是精准模式，那么`View`也是精准模式并且其大小是父容器的剩余空间；如果父容器是最大模式，那么`View`也是最大模式并且其大小不会超过父容器的剩余空间。当`View`的宽/高是`wrap_content`时，不管父容器的模式是精准还是最大化，`View`的模式总是最大化并且大小不能超过父容器的剩余空间。在我们的分析中漏掉了`UNSPECIFIED`模式，那是因为这个模式主要用于系统内部多次Measure的情形，一般来说，我们不需要关注此模式。

#### View的工作流程
##### measure过程
measure过程要分情况来看，如果只是一个原始的`View`，那么通过`measure`方法就完成了其测量过程，如果是一个`ViewGroup`，除了完成自己的测量过程外，还会遍历去调用所有子元素的`measure`方法，各个子元素再递归去执行这个流程。
** View的measure过程 **
`View`的`measure`过程由其`measure`方法来完成，`measure`方法是一个`final`类型的方法，这意味着子类不能重写此方法，在`View`的`measure`方法中会去调用`View`的`onMeasure`方法，因此只需要看`onMeasure`的实现即可：
``` java
protected void onMeasure(int widthMeasureSpec,int heightMeasureSpec) {
    setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(),widthMeasureSpec),getDefaultSize(getSuggestedMinimumHeight(),heightMeasureSpec));
}
public static int getDefaultSize(int size,int measureSpec) {
    int result = size;
    int specMode = MeasureSpec.getMode(measureSpec);
    int specSize = MeasureSpec.getSize(measureSpec);
    switch (specMode) {
        case MeasureSpec.UNSPECIFIED:
            result = size;
        break;
        case MeasureSpec.AT_MOST:
        case MeasureSpec.EXACTLY:
            result = specSize;
        break;
    }
    return result;
}
```
可以看出，`getDefaultSize`这个方法的逻辑很简单，对于我们来说，我们只需要看`AT_MOST`和`EXACTLY`这两种情况。简单地理解，其实`getDefaultSize`返回的大小就是`measureSpec`中的`specSize`，而这个`specSize`就是`View`测量后的大小，这里多次提到测量后的大小，是因为`View`最终的大小是在`layout`阶段确定的，所以这里必须要加以区分，但是几乎所有情况下`View`的测量大小和最终大小是相等的。
至于`UNSPECIFIED`这种情况，一般用于系统内部的测量过程，在这种情况下，`View`的大小为`getDefaultSize`的第一个参数`size`，即宽/高分别为`getSuggestedMinimumWidth`和`getSuggestedMinimumHeight`这两个方法的返回值，看一下它们的源码:
``` java
protected int getSuggestedMinimumWidth() {
    return (mBackground == null) ? mMinWidth : max(mMinWidth,mBackground.getMinimumWidth());
}
protected int getSuggestedMinimumHeight() {
    return (mBackground == null) ? mMinHeight : max(mMinHeight,mBackground.getMinimumHeight());
}
```
这里只分析`getSuggestedMinimumWidth`方法的实现，`getSuggestedMinimumHeight`和它的实现原理是一样的。从`getSuggestedMinimumWidth`的代码可以看出，如果`View`没有设置背景，那么`View`的宽度为`mMinWidth`，而`mMinWidth`对应于`android:minWidth`这个属性所指定的值，因此`View`的宽度即为`android:minWidth`属性所指定的值。这个属性如果不指定，那么`mMinWidth`则默认为0；如果`View`指定了背景，则`View`的宽度为`max(mMinWidth,mBackground.getMinimumWidth())`。`mMinWidth`的含义我们已经知道了，那么`mBackground.getMinimumWidth()`是什么呢？我们看一下`Drawable`的`getMinimumWidth`方法，如下所示:
``` java
public int getMinimumWidth() {
    final int intrinsicWidth = getIntrinsicWidth();
    return intrinsicWidth > 0 ? intrinsicWidth : 0;
}
```
可以看出，`getMinimumWidth`返回的就是`Drawable`的原始宽度，前提是这个`Drawable`有原始宽度，否则就返回0。
这里再总结一下`getSuggestedMinimumWidth`的逻辑：如果`View`没有设置背景，那么返回`android:minWidth`这个属性所指定的值，这个值可以为0；如果`View`设置了背景，则返回`android:minWidth`和背景的最小宽度这两者中的最大值，`getSuggestedMinimumWidth`和`getSuggestedMinimumHeight`的返回值就是`View`在`UNSPECIFIED`情况下的测量宽/高。
从`getDefaultSize`方法的实现来看，`View`的宽/高由`specSize`决定，所以我们可以得出如下结论：直接继承`View`的自定义控件需要重写`onMeasure`方法并设置`wrap_content`时的自身大小，否则在布局中使用`wrap_content`就相当于使用`match_parent`。
从上述代码中我们知道，如果`View`在布局中使用`wrap_content`，那么它的`specMode`是`AT_MOST`模式，在这种模式下，它的宽/高等于`specSize`；这种情况下`View`的`specSize`是p`arentSize`，而`parentSize`是父容器中目前可以使用的大小，也就是父容器当前剩余的空间大小。很显然，`View`的宽/高就等于父容器当前剩余的空间大小，这种效果和在布局中使用`match_parent`完全一致。如何解决这个问题呢？也很简单，代码如下所示：
``` java
protected void onMeasure(int widthMeasureSpec,int heightMeasureSpec) {
    super.onMeasure(widthMeasureSpec,heightMeasureSpec);
    int widthSpecMode = MeasureSpec.getMode(widthMeasureSpec);
    int widthSpecSize = MeasureSpec.getSize(widthMeasureSpec);
    int heightSpecMode = MeasureSpec.getMode(heightMeasureSpec);
    int heightSpecSize = MeasureSpec.getSize(heightMeasureSpec);
    if (widthSpecMode == MeasureSpec.AT_MOST && heightSpecMode == MeasureSpec.AT_MOST) {
        setMeasuredDimension(mWidth,mHeight);
    } else if (widthSpecMode == MeasureSpec.AT_MOST) {
        setMeasuredDimension(mWidth,heightSpecSize);
    } else if (heightSpecMode == MeasureSpec.AT_MOST) {
        setMeasuredDimension(widthSpecSize,mHeight);
    }
}
```
在上面的代码中，我们只需要给`View`指定一个默认的内部宽/高（`mWidth`和`mHeight`），并在`wrap_content`时设置此宽/高即可。对于非`wrap_content`情形，我们沿用系统的测量值即可，至于这个默认的内部宽/高的大小如何指定，这个没有固定的依据，根据需要灵活指定即可。如果查看`TextView`、`ImageView`等的源码就可以知道，针对`wrap_content`情形，它们的`onMeasure`方法均做了特殊处理。

** ViewGroup的measure过程 **
对于`ViewGroup`来说，除了完成自己的`measure`过程以外，还会遍历去调用所有子元素的`measure`方法，各个子元素再递归去执行这个过程。和`View`不同的是，`ViewGroup`是一个抽象类，因此它没有重写`View`的`onMeasure`方法，但是它提供了一个叫`measureChildren`的方法，如下所示。
``` java
protected void measureChildren(int widthMeasureSpec,int heightMeasureSpec) {
    final int size = mChildrenCount;
    final View[] children = mChildren;
    for (int i = 0; i < size; ++i) {
        final View child = children[i];
        if ((child.mViewFlags & VISIBILITY_MASK) != GONE) {
            measureChild(child,widthMeasureSpec,heightMeasureSpec);
        }
    }
}
```
从上述代码来看，`ViewGroup`在`measure`时，会对每一个子元素进行`measure`，`measureChild`这个方法的实现也很好理解，如下所示
``` java
protected void measureChild(View child,int parentWidthMeasureSpec,int parentHeightMeasureSpec) {
    final LayoutParams lp = child.getLayoutParams();
    final int childWidthMeasureSpec = getChildMeasureSpec(parentWidth-MeasureSpec,mPaddingLeft + mPaddingRight,lp.width);
    final int childHeightMeasureSpec = getChildMeasureSpec(parentHeight-MeasureSpec,mPaddingTop + mPaddingBottom,lp.height);
    child.measure(childWidthMeasureSpec,childHeightMeasureSpec);
}
```
很显然，`measureChild`的思想就是取出子元素的`LayoutParams`，然后再通过`getChildMeasureSpec`来创建子元素的`MeasureSpec`，接着将`MeasureSpec`直接传递给`View`的`measure`方法来进行测量。我们知道，`ViewGroup`并没有定义其测量的具体过程，这是因为`ViewGroup`是一个抽象类，其测量过程的`onMeasure`方法需要各个子类去具体实现，比如`LinearLayout`、`RelativeLayout`等。

##### layout过程
`Layout`的作用是`ViewGroup`用来确定子元素的位置，当`ViewGroup`的位置被确定后，它在`onLayout`中会遍历所有的子元素并调用其`layout`方法，在`layout`方法中`onLayout`方法又会被调用。`Layout`过程和`measure`过程相比就简单多了，`layout`方法确定`View`本身的位置，而`onLayout`方法则会确定所有子元素的位置，先看`View`的`layout`方法，如下所示。
``` java
public void layout(int l,int t,int r,int b) {
    if ((mPrivateFlags3 & PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT) != 0) {
        onMeasure(mOldWidthMeasureSpec,mOldHeightMeasureSpec);
        mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
    }
    int oldL = mLeft;
    int oldT = mTop;
    int oldB = mBottom;
    int oldR = mRight;
    boolean changed = isLayoutModeOptical(mParent) ?setOpticalFrame(l,t,r,b) : setFrame(l,t,r,b);
    if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
        onLayout(changed,l,t,r,b);
        mPrivateFlags &= ~PFLAG_LAYOUT_REQUIRED;
        ListenerInfo li = mListenerInfo;
        if (li != null && li.mOnLayoutChangeListeners != null) {
            ArrayList<OnLayoutChangeListener> listenersCopy =(ArrayList<OnLayoutChangeListener>)li.mOnLayout-ChangeListeners.clone();
            int numListeners = listenersCopy.size();
            for (int i = 0; i < numListeners; ++i) {
                listenersCopy.get(i).onLayoutChange(this,l,t,r,b,oldL,oldT,oldR,oldB);
            }
        }
    }
    mPrivateFlags &= ~PFLAG_FORCE_LAYOUT;
    mPrivateFlags3 |= PFLAG3_IS_LAID_OUT;
}
```
`layout`方法的大致流程如下：首先会通过`setFrame`方法来设定`View`的四个顶点的位置，即初始化`mLeft`、`mRight`、`mTop`和`mBottom`这四个值，`View`的四个顶点一旦确定，那么`View`在父容器中的位置也就确定了；接着会调用`onLayout`方法，这个方法的用途是父容器确定子元素的位置，和`onMeasure`方法类似，`onLayout`的具体实现同样和具体的布局有关，所以`View`和`ViewGroup`均没有真正实现`onLayout`方法。接下来，我们可以看一下`LinearLayout`的`onLayout`方法，如下所示。
``` java
protected void onLayout(boolean changed,int l,int t,int r,int b) {
    if (mOrientation == VERTICAL) {
        layoutVertical(l,t,r,b);
    } else {
        layoutHorizontal(l,t,r,b);
    }
}
```
`LinearLayout`中`onLayout`的实现逻辑和`onMeasure`的实现逻辑类似，这里选择`layoutVertical`继续讲解，为了更好地理解其逻辑，这里只给出了主要的代码：
``` java
void layoutVertical(int left,int top,int right,int bottom) {
    ......
    final int count = getVirtualChildCount();
    for (int i = 0; i < count; i++) {
        final View child = getVirtualChildAt(i);
        if (child == null) {
            childTop += measureNullChild(i);
        } else if (child.getVisibility() != GONE) {
            final int childWidth = child.getMeasuredWidth();
            final int childHeight = child.getMeasuredHeight();
            final LinearLayout.LayoutParams lp =(LinearLayout.LayoutParams) child.getLayoutParams();
            ......
            if (hasDividerBeforeChildAt(i)) {
                childTop += mDividerHeight;
            }
            childTop += lp.topMargin;
            setChildFrame(child,childLeft,childTop + getLocationOffset(child),childWidth,childHeight);
            childTop += childHeight + lp.bottomMargin + getNextLocation-Offset(child);
            i += getChildrenSkipCount(child,i);
        }
    }
}
```
这里分析一下`layoutVertical`的代码逻辑，可以看到，此方法会遍历所有子元素并调用`setChildFrame`方法来为子元素指定对应的位置，其中`childTop`会逐渐增大，这就意味着后面的子元素会被放置在靠下的位置，这刚好符合竖直方向的`LinearLayout`的特性。至于`setChildFrame`，它仅仅是调用子元素的`layout`方法而已，这样父元素在`layout`方法中完成自己的定位以后，就通过`onLayout`方法去调用子元素的`layout`方法，子元素又会通过自己的`layout`方法来确定自己的位置，这样一层一层地传递下去就完成了整个`View`树的`layout`过程。`setChildFrame`方法的实现如下所示。
``` java
private void setChildFrame(View child,int left,int top,int width,int height) {
    child.layout(left,top,left + width,top + height);
}
```
我们注意到，setChildFrame中的width和height实际上就是子元素的测量宽/高，从下面的代码可以看出这一点：
``` java
    final int childWidth = child.getMeasuredWidth();
    final int childHeight = child.getMeasuredHeight();
    setChildFrame(child,childLeft,childTop + getLocationOffset(child),childWidth,childHeight);
```

而在`layout`方法中会通过`setFrame`去设置子元素的四个顶点的位置，在`setFrame`中有如下几句赋值语句，这样一来子元素的位置就确定了：
``` java
    mLeft = left;
    mTop = top;
    mRight = right;
    mBottom = bottom;
```
`View`的测量宽/高和最终/宽高有什么区别？这个问题可以具体为：`View`的`getMeasuredWidth`和`getWidth`这两个方法有什么区别，至于`getMeasuredHeight`和`getHeight`的区别和前两者完全一样。为了回答这个问题，首先，我们看一下getwidth和getHeight这两个方法的具体实现：
``` java
public final int getWidth() {
    return mRight -mLeft;
}
public final int getHeight() {
    return mBottom -mTop;
}
``` 

从`getWidth`和`getHeight`的源码再结合`mLeft`、`mRight`、`mTop`和`mBottom`这四个变量的赋值过程来看，`getWidth`方法的返回值刚好就是`View`的测量宽度，而`getHeight`方法的返回值也刚好就是`View`的测量高度。经过上述分析，现在我们可以回答这个问题了：在`View`的默认实现中，`View`的测量宽/高和最终宽/高是相等的，只不过测量宽/高形成于`View`的`measure`过程，而最终宽/高形成于`View`的`layout`过程，即两者的赋值时机不同，测量宽/高的赋值时机稍微早一些。因此，在日常开发中，我们可以认为`View`的`测量宽/高`就`等于``最终宽/高`，但是的确存在某些特殊情况会导致两者不一致.

##### draw过程
Draw过程就比较简单了，它的作用是将View绘制到屏幕上面。View的绘制过程遵循
如下几步：
* 绘制背景background.draw(canvas)。
* 绘制自己（onDraw）。
* 绘制children（dispatchDraw）。
* 绘制装饰（onDrawScrollBars）。
这一点通过draw方法的源码可以明显看出来，如下所示。

``` java
    public void draw(Canvas canvas) {
    final int privateFlags = mPrivateFlags;
    final boolean dirtyOpaque = (privateFlags & PFLAG_DIRTY_MASK) == PFLAG_DIRTY_OPAQUE &&(mAttachInfo == null || !mAttachInfo.mIgnoreDirtyState);
    mPrivateFlags = (privateFlags & ~PFLAG_DIRTY_MASK) | PFLAG_DRAWN;
    /*
    * Draw traversal performs several drawing steps which must be executed
    * in the appropriate order:
    *
    * 1. Draw the background
    * 2. If necessary,save the canvas' layers to prepare for fading
    * 3. Draw view's content
    * 4. Draw children
    * 5. If necessary,draw the fading edges and restore layers
    * 6. Draw decorations (scrollbars for instance)
    */
    // Step 1,draw the background,if needed
    int saveCount;
    if (!dirtyOpaque) {
        drawBackground(canvas);
    }
    // skip step 2 & 5 if possible (common case)
    final int viewFlags = mViewFlags;
    boolean horizontalEdges = (viewFlags & FADING_EDGE_HORIZONTAL) != 0;
    boolean verticalEdges = (viewFlags & FADING_EDGE_VERTICAL) != 0;
    if (!verticalEdges && !horizontalEdges) {
        // Step 3,draw the content
        if (!dirtyOpaque) onDraw(canvas);
            // Step 4,draw the children
        dispatchDraw(canvas);
        // Step 6,draw decorations (scrollbars)
        onDrawScrollBars(canvas);
        if (mOverlay != null && !mOverlay.isEmpty()) {
            mOverlay.getOverlayView().dispatchDraw(canvas);
        }
        // we're done...
        return;
    }
    ...
}
```
View绘制过程的传递是通过dispatchDraw来实现的，dispatchDraw会遍历调用所有子元素的draw方法，如此draw事件就一层层地传递了下去。View有一个特殊的方法setWillNotDraw，先看一下它的源码，如下所示。
``` java
    /**
    * If this view doesn't do any drawing on its own,set this flag to
    * allow further optimizations. By default,this flag is not set on
    * View,but could be set on some View subclasses such as ViewGroup.
    *
    * Typically,if you override {@link #onDraw(android.graphics.Canvas)}
    * you should clear this flag.
    *
    * @param willNotDraw whether or not this View draw on its own
    */
    public void setWillNotDraw(boolean willNotDraw) {
        setFlags(willNotDraw ? WILL_NOT_DRAW : 0,DRAW_MASK);
    }
```
从`setWillNotDraw`这个方法的注释中可以看出，如果一个`View`不需要绘制任何内容，那么设置这个标记位为`true`以后，系统会进行相应的优化。默认情况下，`View`没有启用这个优化标记位，但是`ViewGroup`会默认启用这个优化标记位。这个标记位对实际开发的意义是：当我们的自定义控件继承于`ViewGroup`并且本身不具备绘制功能时，就可以开启这个标记位从而便于系统进行后续的优化。当然，当明确知道一个`ViewGroup`需要通过`onDraw`来绘制内容时，我们需要显式地关闭`WILL_NOT_DRAW`这个标记位。

----
以上