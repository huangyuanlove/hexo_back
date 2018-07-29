---
title: View绘制过程
date: 2018-07-29 22:35:46
tags: [Android]
---
`ViewRoot`对应于`ViewRootImpl`类，它是连接`WindowManager`和`DecorView`的纽带，`View`的三大流程均是通过`ViewRoot`来完成的。在`ActivityThread`中，当`Activity`对象被创建完毕后，会将`DecorView`添加到`Window`中，同时会创建`ViewRootImpl`对象，并将`ViewRootImpl`对象和`DecorView`建立关联，`View`的绘制流程是从`ViewRoot`的`performTraversals`方法开始的，它经过`measure`、`layout`和`draw`三个过程才能最终将一个`View`绘制出来，其中`measure`用来测量View的宽和高，`layout`用来确定View在父容器中的放置位置，而`draw`则负责将View绘制在屏幕上。

<!--more-->
`performTraversals`会依次调用`performMeasure`、p`erformLayout`和`performDraw`三个方法，这三个方法分别完成顶级`View`的`measure`、`layout`和`draw`这三大流程，其中在`performMeasure`中会调用`measure`方法，在`measure`方法中又会调用`onMeasure`方法，在`onMeasure`方法中则会对所有的子元素进行`measure`过程，这个时候`measure`流程就从父容器传递到子元素中了，这样就完成了一次`measure`过程。接着子元素会重复父容器的`measure`过程，如此反复就完成了整个`View`树的遍历。同理，`performLayout`和`performDraw`的传递流程和`performMeasure`是类似的，唯一不同的是，`performDraw`的传递过程是在`draw`方法中通过`dispatchDraw`来实现的，不过这并没有本质区别。

`measure`过程决定了`View`的宽/高，`Measure`完成以后，可以通过`getMeasuredWidth`和`getMeasuredHeight`方法来获取到`View`测量后的宽/高，在几乎所有的情况下它都等同于`View`最终的宽/高，但是特殊情况除外，这点在本章后面会进行说明。`Layout`过程决定了`View`的四个顶点的坐标和实际的View的宽/高，完成以后，可以通过`getTop`、`getBottom`、`getLeft`和`getRight`来拿到`View`的四个顶点的位置，并可以通过`getWidth`和`getHeight`方法来拿到`View`的最终宽/高。`Draw`过程则决定了`View`的显示，只有`draw`方法完成以后`View`的内容才能呈现在屏幕上。

`DecorView`作为顶级`View`，一般情况下它内部会包含一个竖直方向的`LinearLayout`，在这个`LinearLayout`里面有上下两个部分（具体情况和Android版本及主题有关），上面是标题栏，下面是内容栏。在`Activity`中我们通过`setContentView`所设置的布局文件其实就是被加到内容栏之中的，而内容栏的id是`content`，因此可以理解为Activity指定布局的方法不叫setview而叫`setContentView`，因为我们的布局的确加到了`id`为`content`的`FrameLayout`中。如何得到`content`呢？可以这样：`ViewGroup  content=  findViewById(R.android.id.content)`。如何得到我们设置的`View`呢？可以这样：`content.getChildAt(0)`。同时，通过源码我们可以知道，`DecorView`其实是一个`FrameLayout`，`View`层的事件都先经过`DecorView`，然后才传递给我们的View。