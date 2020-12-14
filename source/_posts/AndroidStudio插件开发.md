---
title: AndroidStudio插件开发
tags: [Android,Plugin]
date: 2020-12-13 22:42:42
keywords: [AndroidStudioPlugin,AS插件开发]
---

写了个类似`Butter Knife`的[开发库](https://github.com/huangyuanlove/AndroidAnnotation)，但是并没有与其配套的AndroidStudio插件，抽时间研究了以下IDEA的api文档，撸了一个对应的插件，[源码在这里](https://github.com/huangyuanlove/AndroidAnnotation-Plugin)

所用到的知识点：

1. 查找文件
2. 解析xml
3. 写文件
IDEA插件开发文档：https://jetbrains.org/intellij/sdk/docs/intro/welcome.html
<!--more-->

#### 创建项目

官方推荐创建gradle项目，这里贴个图，创建过程按照官网叙述的创建就好

https://jetbrains.org/intellij/sdk/docs/tutorials/build_system/prerequisites.html

这里说明一下，如果想要在AndroidStudio中进行debug，阅读一下这个

https://jetbrains.org/intellij/sdk/docs/products/android_studio.html

也就是在项目根目录的的`build.gradle`中配置 `intellij`和`runIde`，具体含义可在网页中找到，这里不再赘述

``` groovy
// See https://github.com/JetBrains/gradle-intellij-plugin/
intellij {
    version '201.8743.12'
    type 'IC'
    plugins = ['android', 'java']
}
runIde {
    // Absolute path to installed target 3.5 Android Studio to use as IDE Development Instance
    // The "Contents" directory is macOS specific.
//    ideDirectory '/Applications/Android Studio.app/Contents' //for mac
//    ideDirectory '/home/huangyuan/androidStudio' //for linux
    ideDirectory 'G:\\AndroidStudio' //for window
}
```



![create_plugin_gradle](/image/Android/AndroidStudioPlugin/create_plugin_gradle.png)

#### 创建类

创建一个继承AnAction 的类，这里创建的方式有两种，一个是直接创建java类，然后再去注册；另外一个就是通过想到直接创建(就像我们创建Activity一样)；

具体可以看这里 https://jetbrains.org/intellij/sdk/docs/tutorials/action_system/working_with_custom_actions.html

这里我们需要解析layout文件(xml文件)并且还要写入文件，所以就直接继承`BaseGenerateAction`，重写其中的两个方法

``` java

    @Override
    public void update(@NotNull AnActionEvent e) {
       // Using the event, evaluate the context, and enable or disable the action.
        e.getPresentation().setEnabledAndVisible(e.getProject() != null);
    }
    @Override
    public void actionPerformed(@NotNull AnActionEvent event) {
        // Using the event, implement an action. For example, create and show a dialog.
    }
```

当工程处于indexing的时候，我们不想让插件生效，可以实现`DumbAware`接口，继续向`actionPerformed`方法中添加逻辑

``` java
    @Override
    public void actionPerformed(@NotNull AnActionEvent event) {
        //获取工程对象，具体信息可以看这里 https://jetbrains.org/intellij/sdk/docs/basics/project_structure.html
        Project project = event.getData(PlatformDataKeys.PROJECT);
        if(project ==null){
            return ;
        }
        Editor editor = event.getData(PlatformDataKeys.EDITOR);
        if(editor ==null){
            return;
        }
        DumbService dumbService = DumbService.getInstance(project);
        if (dumbService.isDumb()) {
            dumbService.showDumbModeNotification("ViewInject plugin is not available during indexing");
            return;
        }
        //这里是我们自己的逻辑
        analyze(project, editor);
    }
```



#### 获取文件

我们可以获取到当前光标所指向的位置，也可以获取当前选中的字符，我们从官方文档中找到我们自己需要的东西：需要看一下PSI(Program Structure Interface),具体信息在这里https://jetbrains.org/intellij/sdk/docs/basics/architectural_overview/psi.html

关键信息在[PSI element](https://jetbrains.org/intellij/sdk/docs/basics/architectural_overview/psi_elements.html) 和 [PSI Files](https://jetbrains.org/intellij/sdk/docs/basics/architectural_overview/psi_files.html)，项目的中的具体逻辑在`GetLayoutFileUtil.java`，这里比较麻烦一些，用到了`Module`和`GlobalSearchScope`这两个类，[具体可以看这里https://jetbrains.org/intellij/sdk/docs/reference_guide/project_model/module.html](https://jetbrains.org/intellij/sdk/docs/reference_guide/project_model/module.html)，就不再抄一遍+翻译了

#### 解析文件

这里我们拿到了对应的`layout.xml`文件对象，一个`PsiFile`对象，调用文件的遍历方法`layoutFile.accept(PsiElementVisitor visitor)`，这里我们传入`XmlRecursiveElementVisitor`实例对象，在解析xml的过程中，我们可能会遇到`<include>`标签，需要继续解析该标签下的xml文件，这里搞个递归。

#### 展示解析内容

解析出来的数据存入ArrayList中，在解析过程中，保存了对应id，并且将id的值转化为对应的字段名字；保存了是是否为自定义的view，如果是类似于`TextView`之类的控件，则

我使用的java gui，也就是java swing组件来展示的

#### 写入文件

PsiClass 和