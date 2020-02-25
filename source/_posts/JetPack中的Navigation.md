---
title: JetPack中的Navigation
tags: [Android]
date: 2020-02-25 14:32:02
keywords: Android,Jetpack,LiveData,LifeCycle,ViewModel,Navigation
---

2018年谷歌I/O 发布了一系列辅助android开发者的实用工具，合称Jetpack，以帮助开发者构建出色的 Android 应用。

这次发布的 Android Jetpack 组件覆盖以下 4 个方面：Architecture、Foundation、Behavior 以及 UI。

该系列博客介绍一下Jetpack中常用组件，本篇介绍LiveData、ViewModel、LifeCycle。最后借助于https://github.com/android/sunflower 来写一个完整的应用

<!--more-->

原文https://developer.android.com/guide/navigation/?hl=zh-cn，就像它的名字一样，用来做导航，可以用来做单页面应用(一个Activity中多个Fragment进行切换)

导航组件由以下三个关键部分组成：

* 导航图：在一个集中位置包含所有导航相关信息的 XML 资源。这包括应用内所有单个内容区域（称为*目标*）以及用户可以通过应用获取的可能路径。
* `NavHost`：显示导航图中目标的空白容器。导航组件包含一个默认 `NavHost` 实现 ([`NavHostFragment`](https://developer.android.com/reference/androidx/navigation/fragment/NavHostFragment?hl=zh-cn))，可显示 Fragment 目标
* `NavController`：在 `NavHost` 中管理应用导航的对象。当用户在整个应用中移动时，`NavController` 会安排 `NavHost` 中目标内容的交换。



### 使用方式

#### 添加依赖

``` groovy
dependencies {
      def nav_version = "2.3.0-alpha01"

      // Java language implementation
      implementation "androidx.navigation:navigation-fragment:$nav_version"
      implementation "androidx.navigation:navigation-ui:$nav_version"

      // Kotlin
      implementation "androidx.navigation:navigation-fragment-ktx:$nav_version"
      implementation "androidx.navigation:navigation-ui-ktx:$nav_version"

      // Dynamic Feature Module Support
      implementation "androidx.navigation:navigation-dynamic-features-fragment:$nav_version"

      // Testing Navigation
      androidTestImplementation "androidx.navigation:navigation-testing:$nav_version"
    }
```

#### 创建导航图

在`res/navigation`下创建一个`Navigation source file`，根节点如下：

``` xml
<?xml version="1.0" encoding="utf-8"?>
<navigation xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto" 
    android:id="@+id/test_navigation">

</navigation>
```

#### 向 Activity 添加 NavHost

导航宿主是 Navigation 组件的核心部分之一。导航宿主是一个空容器，用户在您的应用中导航时，目的地会在该容器中交换进出。

导航宿主必须派生于 [`NavHost`](https://developer.android.com/reference/androidx/navigation/NavHost?hl=zh-cn)。Navigation 组件的默认 `NavHost` 实现 ([`NavHostFragment`](https://developer.android.com/reference/androidx/navigation/fragment/NavHostFragment?hl=zh-cn)) 负责处理 Fragment 目的地的交换。

在Activit的布局文件中添加如下控件

``` xml
    <fragment
        android:id="@+id/fragment_first"
        android:name="androidx.navigation.fragment.NavHostFragment"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        app:defaultNavHost="true"
        app:navGraph="@navigation/test_navi" />
```

请注意以下几点：

- `android:name` 属性包含 `NavHost` 实现的类名称。
- `app:navGraph` 属性将 `NavHostFragment` 与导航图相关联。导航图会在此 `NavHostFragment` 中指定用户可以导航到的所有目的地。
- `app:defaultNavHost="true"` 属性确保您的 `NavHostFragment` 会拦截系统返回按钮。请注意，只能有一个默认 `NavHost`。如果同一布局（例如，双窗格布局）中有多个主机，请务必仅指定一个默认 `NavHost`。

在Activit中设置支持：

``` java
 @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_navigation);
    }


    @Override
    public boolean onSupportNavigateUp() {
        return Navigation.findNavController(this, R.id.fragment_first).navigateUp();
    }
```

#### 添加Fragment

我们添加FragmentA，FragmentB，FragmentC，每个Fragment中只有一个文本按钮

``` java
public class FragmentA extends Fragment {

    public FragmentA() {
    }

    @Override
    public View onCreateView(LayoutInflater inflater, ViewGroup container,
                             Bundle savedInstanceState) {
        View view =
                inflater.inflate(R.layout.fragment_a, container, false);
        view.findViewById(R.id.jump).setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                FragmentBArgs.Builder builder= new  FragmentBArgs.Builder();
                builder.setNum(11111);
                builder.setTitle("FragmentB");
                Navigation.findNavController(v).navigate(R.id.action_fragmentA_to_fragmentB,builder.build().toBundle());
            }
        });

        return view;
    }

}
```

#### 添加导航

在我们前面步骤创建的Navigation Resource文件中

``` xml
<?xml version="1.0" encoding="utf-8"?>
<navigation xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:id="@+id/test_navi"
    app:startDestination="@id/fragmentA">
    <fragment
        android:id="@+id/fragmentA"
        android:name="com.huangyuanlove.androidjetpack.architecture.navigation.FragmentA"
        android:label="fragmentA"
        tools:layout="@layout/fragment_a">
        <action
            android:id="@+id/action_fragmentA_to_fragmentB"
            app:destination="@id/fragmentB"
            app:exitAnim="@android:anim/slide_out_right" />
    </fragment>
    
    <fragment
        android:id="@+id/fragmentB"

        android:name="com.huangyuanlove.androidjetpack.architecture.navigation.FragmentB"
        android:label="fragmentB"
        tools:layout="@layout/fragment_b">
        <argument
            android:name="title"
            android:defaultValue="test"
            app:argType="string" />
        <argument
            android:name="num"
            android:defaultValue="100"
            app:argType="integer" />
        <action
            android:id="@+id/action_fragmentB_to_fragmentC"
            app:destination="@id/fragmentC"
            app:exitAnim="@android:anim/slide_out_right" />
    </fragment>
    <fragment
        android:id="@+id/fragmentC"

        android:name="com.huangyuanlove.androidjetpack.architecture.navigation.FragmentC"
        android:label="fragmentC"
        tools:layout="@layout/fragment_c">
        <action
            android:id="@+id/action_fragmentC_to_fragmentA"
            app:destination="@id/fragmentA"
            app:exitAnim="@android:anim/slide_out_right" />
    </fragment>

</navigation>
```

- **Type** 字段指示在您的源代码中，该目的地是作为 Fragment、Activity 还是其他自定义类实现的。
- **Label** 字段包含该目的地的 XML 布局文件的名称。
- **ID** 字段包含该目的地的 ID，它用于在代码中引用该目的地。
- **Class** 下拉列表显示与该目的地相关联的类的名称。您可以点击此下拉列表，将相关联的类更改为其他目的地类型。

#### 使用

导航到目的地是使用 [`NavController`](https://developer.android.com/reference/androidx/navigation/NavController?hl=zh-cn) 完成的，后者是一个在 `NavHost` 中管理应用导航的对象。每个 `NavHost` 均有自己的相应 `NavController`。可以使用以下方法之一检索 `NavController`：

- [NavHostFragment.findNavController(Fragment)](https://developer.android.com/reference/androidx/navigation/fragment/NavHostFragment?hl=zh-cn#findNavController(android.support.v4.app.Fragment))
- [Navigation.findNavController(Activity, @IdRes int viewId)](https://developer.android.com/reference/androidx/navigation/Navigation?hl=zh-cn#findNavController(android.app.Activity, int))
- [Navigation.findNavController(View)](https://developer.android.com/reference/androidx/navigation/Navigation?hl=zh-cn#findNavController(android.view.View))

比如我们打算在FragmentA中导航到FragmentB，则

``` java
tv.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                Navigation.findNavController(v)
                        .navigate(R.id.action_fragmentA_to_fragmentB);
            }
        });
```

如果我们需要传递参数，可以使用 Safe Args 来确保类型安全。

在顶级`build.gradle`中添加classPath:`classpath "androidx.navigation:navigation-safe-args-gradle-plugin:2.3.0-alpha02"`

在对应module的`build.gradle`中添加plugin:`apply plugin: 'androidx.navigation.safeargs'`

我们在上面的Navigation Resource中的`fragmentB`中添加了三个`argument`节点，插件会帮我们自动生成`FragmentBArgs`类，我们可以这么使用

```java
view.findViewById(R.id.jump).setOnClickListener(new View.OnClickListener() {
    @Override
    public void onClick(View v) {
        FragmentBArgs.Builder builder= new  FragmentBArgs.Builder();
        builder.setNum(11111);
        builder.setTitle("FragmentB");
        Navigation.findNavController(v).navigate(R.id.action_fragmentA_to_fragmentB,builder.build().toBundle());
    }
});
```

#### 遗留问题

尽管通过Navigation方式管理Fragment的代码简洁而且直观，但是有一个非常致命的问题，就是同一个Fragment会被重复创建，比如A到B，B再到C，然后点击返回键返回到B，这时B会被重建，再点击返回键返回到A，A也会被重建，目前还找不到解决方案，或许是我的使用方式不对



----

以上



