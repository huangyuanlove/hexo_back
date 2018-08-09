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
### 版本支持
ConstraintLayout是一个支持库，向前兼容到Android9，以后还会添加更多的新特性。现在公司的产品的最低版本支持都在2.3之上，部分产品最低版本支持保持在4.4之上。这就意味着我们不需要关心最低版本支持的事情。

### 新特性
在使用新特性的时候是不能有循环依赖的，比如相对定位，不能A依赖于B的位置，B依赖C的位置，而C又依赖A的位置
#### Relative positioning
相对定位是ConstraintLayout中最基本的构建方式，也就是一个空间相对于另外一个空间进行位置确定，可以在横向和竖向上进行约束：
> Horizontal Axis: left, right, start and end sides
  Vertical Axis: top, bottom sides and text baseline

如果我们需要让ButtonB在ButtonA的右侧，如下图：
![Relative Positioning Example](/image/Android/ConstraintLayout/relative_positioning_example.png)

在布局文件中只需要：
``` xml
    <Button android:id="@+id/buttonA" ... />
    <Button android:id="@+id/buttonB" ...
            app:layout_constraintLeft_toRightOf="@+id/buttonA" />
```
这就告诉系统 让buttonB的左边约束于buttonA的右边
下面列出了所有可用的约束方式：

* layout_constraintLeft_toLeftOf
* layout_constraintLeft_toRightOf
* layout_constraintRight_toLeftOf
* layout_constraintRight_toRightOf
* layout_constraintTop_toTopOf
* layout_constraintTop_toBottomOf
* layout_constraintBottom_toTopOf
* layout_constraintBottom_toBottomOf
* layout_constraintBaseline_toBaselineOf
* layout_constraintStart_toEndOf
* layout_constraintStart_toStartOf
* layout_constraintEnd_toStartOf
* layout_constraintEnd_toEndOf
  
各约束位置如下：
![relative_position_constraint](/image/Android/ConstraintLayout/relative_position_constraint.png)
上面这些约束关系全部都是本身相对于另外一个控件(使用@id方式引用另外控件)或者父布局(使用parent方式引用父控件)进行约束

#### Margins

![Relative Positioning Margins](/image/Android/ConstraintLayout/relative_positioning_margins.png)
这里的外边距和其他布局方式的外边距一样，不能是负数，属性如下：
* android:layout_marginStart
* android:layout_marginEnd
* android:layout_marginLeft
* android:layout_marginTop
* android:layout_marginRight
* android:layout_marginBottom

添加的一个新属性是 `maiginGone`,当一个约束目标的可见性为GONE的时候(View.GONE)，可以改变当前控件的外边距，比如B是相对于A进行约束，当A不可见的时候，可以改变B的外边距，也就是B的外边距可以根据Ade可见性设置不同的值，属性如下
* layout_goneMarginStart
* layout_goneMarginEnd
* layout_goneMarginLeft
* layout_goneMarginTop
* layout_goneMarginRight
* layout_goneMarginBottom

#### Centering positioning and bias
** Centering Positioning **
如果对一个控件的左右(上下)都添加的约束，那么ConstraintLayout的表现就像有两个大小相等方向相反的力在拉这个控件一个样，比如
``` xml
<android.support.constraint.ConstraintLayout ...>
    <Button android:id="@+id/button" ...
        app:layout_constraintLeft_toLeftOf="parent"
        app:layout_constraintRight_toRightOf="parent/>
</>
```
表现如下：
![Centering Positioning](/image/Android/ConstraintLayout/centering_positioning.png)
这样就会产生居中效果，如果子控件和父控件的尺寸相同，这写属性就没有意义了

** bias **
当遇到上面这种约束的时候，我们可以使用`bias`属性让控件偏向于哪一个方向,属性如下：
* layout_constraintHorizontal_bias
* layout_constraintVertical_bias
例如，如下代码：
``` xml
<android.support.constraint.ConstraintLayout ...>
    <Button android:id="@+id/button" ...
        app:layout_constraintHorizontal_bias="0.3"
        app:layout_constraintLeft_toLeftOf="parent"
        app:layout_constraintRight_toRightOf="parent/>
</>
```
表现如下：
![Centering Positioning with Bias](/image/Android/ConstraintLayout/centering_positioning_with_bias.png)



#### Circular positioning (Added in 1.1)
这个属性是1.1版本添加进去的，可以使用`angle`和`distance`来约束一个控件的中心点和另外一个空间的中心点的位置关系，这样就可以把空间定位在一个圆上，可用属性如下：

* **layout_constraintCircle ** : references another widget id
* **layout_constraintCircleRadius ** : the distance to the other widget center
* **layout_constraintCircleAngle ** : which angle the widget should be at (in degrees, from 0 to 360)

示例如下：
``` xml
<Button android:id="@+id/buttonA" ... />
<Button android:id="@+id/buttonB" ...
      app:layout_constraintCircle="@+id/buttonA"
      app:layout_constraintCircleRadius="100dp"
      app:layout_constraintCircleAngle="45" />
```
表现如下：
![Circular Positioning](/image/Android/ConstraintLayout/circular_positioning.png)

#### Visibility behavior
ConstraintLayout对于被标记为GONE的控件有特殊的处理。一般布局中，GONE控件是不会展示在界面上并且不再属于布局的一部分，但是在布局计算上，ConstraintLayout和传统布局有很大的区别
1. 传统布局下，GONE控件会被认为大小是0，也就是一个点
2. 在ConstraintLayout中其大小仍然按照可见大小计算，但是其外边距为0

![Visibility Behavior](/image/Android/ConstraintLayout/visibility_behavior.png)


#### Dimension constraints
##### Minimum dimensions on ConstraintLayout
可以像普通控件一样设置最小最大尺寸,属性如下：
* ** android:minWidth ** set the minimum width for the layout
* ** android:minHeight ** set the minimum height for the layout
* ** android:maxWidth ** set the maximum width for the layout
* ** android:maxHeight ** set the maximum height for the layout

这些属性当ConstraintLayout的宽高为`WRAP_CONTENT`时有效。

##### Widgets dimension constraints
可以通过`android:layout_width`和`android:layout_height`设置控件的尺寸，有三种方式：
* 固定值
* WRAP_CONTENT
* 0dp, 相当于`MATCH_CONSTRAINT`

前两种方式和普通布局表现出来的行为一样。最后一种会通过约束来重新设置控件尺寸，如果设置了margin，在布局计算的时候也会被考虑进去。
 ![Dimension Constraints](/image/Android/ConstraintLayout/dimension_constraints.png)
上图中的a是wrap_content,b是0dp，c是设置了margin的0dp。需要注意的是，在ConstraintLayout中，MATCH_PARENT是不推荐使用的。

##### WRAP_CONTENT:enforcing constraints (Added in 1.1)
如果控件实际尺寸超过了约束的尺寸，那么约束就会失效，这时候可以添加如下属性来限制：
* app:layout_constrainedWidth=”true|false”
* app:layout_constrainedHeight=”true|false”

将B控件约束于A控件和父控件的中间，尺寸都为`wrap_content`
![enforcing constraints](/image/Android/ConstraintLayout/wrap_content_enforcing_constraints1.png)
这时候如果将B控件填充很长的文件，那么B控件的左侧则会突破约束，和A控件的中心对齐，如果我们不想要这种方式，还是要求B的左侧和Ade右侧对齐，则可以天剑
`layout_constrainedWidth="true"`属性进行约束，实例如下：
![约束失效](/image/Android/ConstraintLayout/wrap_content_enforcing_constraints2.png)
![添加强制约束属性](/image/Android/ConstraintLayout/wrap_content_enforcing_constraints3.png)

##### MATCH_CONSTRAINT dimensions (Added in 1.1)

当控件的尺寸被设置为`MATCH_CONSTRAINT`时，默认的行为是占据所有的剩余空间，可以使用如下属性来更改此行为：
* ** layout_constraintWidth_min ** 和 ** layout_constraintHeight_min ** : will set the minimum size for this dimension
* ** layout_constraintWidth_max ** 和 ** layout_constraintHeight_max ** : will set the maximum size for this dimension
* ** layout_constraintWidth_percent ** 和 ** layout_constraintHeight_percent ** : will set the size of this dimension as a percentage of the parent

##### Percent dimension

To use percent, you need to set the following:
想要使用百分比布局，需要设置如下属性：
1. 控件宽高设置为 `MATCH_CONSTRAINT` (0dp)
2. `app:layout_constraintWidth_default`属性值设置为`percent` 
3. 设置 `layout_constraintWidth_percent`或者`layout_constraintHeight_percent`属性值(0-1之间)

下面的TextView控件将占据剩余宽度的50%和剩余高度的50%,示例：
``` xml
<TextView
        android:id="@+id/textView6"
        android:layout_width="0dp"
        android:layout_height="0dp"
        android:background="@color/colorAccent"
        app:layout_constraintHeight_default="percent"
        app:layout_constraintHeight_percent="0.5"
        app:layout_constraintWidth_default="percent"
        app:layout_constraintWidth_percent="0.5" />
```
##### Ratio


#### Chains
#### Virtual Helpers objects
#### Optimizer