---
title: ConstraintLayout
date: 2018-08-09 10:40:53
tags: [Android]
keywords: ConstraintLayout
photos: 
  - /image/Android/ConstraintLayout/class_summary.png
---
https://developer.android.google.cn/reference/android/support/constraint/ConstraintLayout
A ConstraintLayout is a ViewGroup which allows you to position and size widgets in a flexible way.

<!--more-->
#### 版本支持
ConstraintLayout是一个支持库，向前兼容到Android9，以后还会添加更多的新特性。现在公司的产品的最低版本支持都在2.3之上，部分产品最低版本支持保持在4.4之上。这就意味着我们不需要关心最低版本支持的事情。

#### 新特性
在使用新特性的时候是不能有循环依赖的，比如相对定位，不能A依赖于B的位置，B依赖C的位置，而C又依赖A的位置
##### Relative positioning
相对定位是ConstraintLayout中最基本的构建方式，也就是一个空间相对于另外一个空间进行位置确定，可以在横向和竖向上进行约束：
> Horizontal Axis: left, right, start and end sides
  Vertical Axis: top, bottom sides and text baseline

如果我们需要让ButtonB在ButtonA的右侧，如下图：
![Relative Positioning Example](/image/Android/COnstraintLayout/relative_positioning_example.png)
在布局文件中只需要：
``` xml
    <Button android:id="@+id/buttonA" ... />
    <Button android:id="@+id/buttonB" ...
            app:layout_constraintLeft_toRightOf="@+id/buttonA" />
```
* Margins
* Centering positioning
* Circular positioning
* Visibility behavior
* Dimension constraints
* Chains
* Virtual Helpers objects
* Optimizer