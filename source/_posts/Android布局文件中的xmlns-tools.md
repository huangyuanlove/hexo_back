---
title: 'Android布局文件中的xmlns:tools'
date: 2018-01-04 14:12:34
tags: [Android]
---
在使用AndroidStudio创建布局文件的时候，跟布局下总是有如下代码：
``` xml
<RootTag xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    tools:context="***.***Activity" >
```
于是在官网查了一下这俩货是干嘛用的，下面是自己的翻译+实践
<!--more-->
`xmlns`的全称是`xmlnamespace`,和c++中`namespace`差不多，都是为了解决命名上的冲突问题。
在Android布局文件中，常见的`xmlns`大概有三个（你可以随意命名，把tools改成bug也可以，只要对应使用tools的地方都改成bug，说白了，它只是个变量而已）。
``` xml
xmlns:android="http://schemas.android.com/apk/res/android"
xmlns:tools="http://schemas.android.com/tools"
xmlns:app="http://schemas.android.com/apk/res-auto"
```
###### android
用于Android系统定义的一些属性
###### app
用于我们自定义的一些属性
###### tools
给IDE或者预览界面用的，当打包编译时并不会包含在apk中。
#### tools可以干什么
先看官网介绍 https://developer.android.com/studio/write/tool-attributes.html
> Android Studio supports a variety of XML attributes in the tools namespace that enable design-time features (such as which layout to show in a fragment) or compile-time behaviors (such as which shrinking mode to apply to your XML resources). When you build your app, the build tools remove these attributes so there is no effect on your APK size or runtime behavior.

大致意思就是说使用`tools`后面的属性不会再编译时存在，只存在于设计时的预览，不会影响apk的体积，

##### Error handling attributes
影响Lint提示的属性主要有下面三种：
>**tools:ignore**
Intended for: Any element
Used by: Lint

> **tools:targetApi**
Intended for: Any element
Used by: Lint

>**tools:locale**
Intended for: resources
Used by: Lint, Android Studio editor

`ignore`属性是告诉Lint忽略xml中的某些警告，比如我们在`ImageView`或者`ImageButton`中没有写`contentDescription`属性，Lint会有警告：`Missing contentDescription attribute on image`,原因是这个属性是提供无障碍阅读的，没有这个属性的话`Screen Reader`无法正常工作，但是有些特定分类的软件是不考虑这些东西的。虽然这种警告无所谓，但是对于要求代码中不能有warning的公司来说，这是不可取的，我们可以用`ignore`属性来忽略这个警告(如果你说你改了Lint的警告级别，当我没说)：
``` xml
<ImageView
android:layout_width="wrap_content"
android:layout_height="wrap_content"
android:layout_marginStart="@dimen/margin_main"
android:layout_marginTop="@dimen/margin_main"
android:scaleType="center"
android:src="@drawable/divider"
tools:ignore="contentDescription" />
```
`targetApi`和代码中的注解`@RequiresApi(api = Build.VERSION_CODES.JELLY_BEAN_MR2)`差不多。假设配置文件中的最小sdkLevel为11，而布局文件中使用了21的控件比如`GridLayout`,Lint会有警告，为了消除这个警告，可以这么写：
``` xml
<GridLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    tools:targetApi="14" >
```
`locale`是告诉Lint和AndroidStudio editor默认的是什么语言，如果不指定的话，默认是英语。可以把这个属性添加到`values/strings.xml`中：
``` xml
<resources xmlns:tools="http://schemas.android.com/tools"
    tools:locale="es"/>
```
##### Design-time view attributes(设计时试图属性)
###### tools: instead of android
**Intended for:** *`View`*
**Used by:**  *Android Studio layout editor*

我们有一个`TextView`，需要显示从网络获取到的文件，控件大小是`wrap_content`,但是`TextView`中没有文字的话在预览界面中是看不到的，大部分同学可能会使用`android:text=XXXX`这个属性，调整好布局之后再把文字删除。一两个控件还好说，控件多了指不定哪个控件就忘记删除文本了，我们可以使用`tools`这个东西：
``` xml
<TextView
    android:id="@+id/unlock_bike"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:background="#12345678"
    android:paddingBottom="10dp"
    android:paddingTop="10dp"
    tools:text="测试用例" />
```
这时我们在IDE预览界面是可以看到文字的，打包成apk运行在手机上时是看不到的。
总之，`tools`可以告诉`Android Studio`哪些属性在运行的时候是被忽略的，只在设计布局的时候有效。基本上原生控件的属性都可以这么使用。
###### tools:context
**Intended for:** *Any root View*
**Used by:**  *Lint, Android Studio layout editor*
这个属性告诉IDE当前布局和哪个activity相关联，在预览界面使用关联Activity的主题展示。
``` xml
<android.support.constraint.ConstraintLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    tools:context=".MainActivity" >
```
###### tools:itemCount
**Intended for:** *RecyclerView*
**Used by:** *Android Studio layout editor*
这个属性告诉编辑器在预览窗口展示多少个列表项
``` xml
<android.support.v7.widget.RecyclerView
    android:id="@+id/recyclerView"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:itemCount="3"/>
```
###### tools:layout
**Intended for:** *`fragment`*
**Used by:** *Android Studio layout editor*
这个属性告诉编辑器fragment中显示哪个布局文件
``` xml
<fragment android:name="com.example.master.ItemListFragment"
    tools:layout="@layout/list_content" />
```
###### tools:listitem / tools:listheader / tools:listfooter
**Intended for:** *`AdapterView` (and subclasses like `ListView`)*
**Used by:** *Android Studio layout editor*
看属性名字就能猜出来了，预览界面显示列表的头部，底部和列表项布局
``` xml
<ListView xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:id="@android:id/list"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:listitem="@layout/sample_list_item"
    tools:listheader="@layout/sample_list_header"
    tools:listfooter="@layout/sample_list_footer" />
```
###### tools:showIn
**Intended for:** *Any root `View` in a layout that's referred to by an `include`*
**Used by:** *Android Studio layout editor*
此属性可以通过指向使用此布局包含的布局，因此您可以预览(和编辑)这个文件，因为它嵌入在其父布局时出现
``` xml
<TextView xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:text="@string/hello_world"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    tools:showIn="@layout/activity_main" />
```
###### tools:menu
**Intended for:**  *Any root `View`*
**Used by:** *Android Studio layout editor*
此属性指定菜单应该显示在应用程序栏的布局预览。该值可以是一个或多个菜单id，由逗号分隔
``` xml
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:orientation="vertical"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:menu="menu1,menu2" />
```
###### tools:minValue / tools:maxValue
**Intended for:** *`NumberPicker`*
**Used by:** *Android Studio layout editor*
设置NumberPicker的最大值和最小值
``` xml
<NumberPicker xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:id="@+id/numberPicker"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    tools:minValue="0"
    tools:maxValue="10" />
```
###### tools:openDrawer
**Intended for:** *`DrawerLayout`*
**Used by:** *Android Studio layout editor*
设置`DrawerLayout`在预览窗口的打开位置

|Constant|Value|Description|
|:-------------:|:-------------:|:-----:|
|end|800005|Push object to the end of its container, not changing its size. |
|left|3|Push object to the left of its container, not changing its size.|
|right|5|Push object to the right of its container, not changing its size.|
|start|800003|Push object to the beginning of its container, not changing its size.|

``` xml
<android.support.v4.widget.DrawerLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:id="@+id/drawer_layout"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:openDrawer="start" />
```
下面的不想翻译了，链接在下面
https://developer.android.com/studio/write/tool-attributes.html#resource_shrinking_attributes
自己撸吧

----
以上