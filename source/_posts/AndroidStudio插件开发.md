---
title: AndroidStudio插件开发
tags: [Android,Plugin]
date: 2020-12-13 22:42:42
keywords: [AndroidStudioPlugin,AS插件开发]
---

写了个类似`Butter Knife`的[开发库](https://github.com/huangyuanlove/AndroidAnnotation)，但是并没有与其配套的AndroidStudio插件，抽时间研究了以下IDEA的api文档，撸了一个对应的插件，[源码在这里](https://github.com/huangyuanlove/AndroidAnnotation-Plugin)

代码参考https://github.com/avast/android-butterknife-zelezny

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

解析出来的数据存入ArrayList中，在解析过程中，保存了对应id、判断是否引用了android name space中id、将id的值转化为对应的字段名字、保存了是是否为自定义的view等信息

``` java
    public String id;
    public boolean isAndroidNS = false;
    public String nameFull; // element name with package
    public String name; // element name
    public String fieldName; // name of variable
    public boolean isValid = false;
    public boolean used = true;
    public boolean isClick = true;
```

展示解析内容使用`javax.swing`组件，这个也没什么好说的。

![show_android_annotation_info](/image/Android/AndroidStudioPlugin/AndroidAnnotation.png)

在展示面板上提供的全选功能；提供了生成代码的两种格式

``` java
@BindView(idStr = "xxxx") //可在library、application中使用
@BindView(id = R.id.xxx) //仅在application中使用
```

因为在library中生成的R文件中的变量不是final类型，并且application中的R文件变量，在gradle plugin  5.0之后也不再是final的，所以建议使用idStr的方式，也是默认生成的代码

#### 写入文件

为了方便，写入文件的时候使用的是`PsiClass`对象进行操作的，[源码在这里](https://upsource.jetbrains.com/idea-ce/file/idea-ce-4b94ba01122752d7576eb9d69638b6e89d1671b7/java/java-psi-api/src/com/intellij/psi/PsiClass.java)，至于如何操作PsiFile，[可以看这里](https://jetbrains.org/intellij/sdk/docs/basics/architectural_overview/psi.html)。写入文件的过程，看起来个使用`javapoet`差不多，[javapoet可以看这里，github上直接搜索即可](https://github.com/square/javapoet)

``` java
private void generateClick() {
  for (ElementBean element : mElements) {
    if (element.isClick) {
      StringBuilder method = new StringBuilder();
      method.append("@ClickResponder(" + element.getGenerateValue(generateId) + ")");
      method.append("public void onClick" + Utils.capitalize(element.fieldName) + " (View v) {}");
      mClass.add(mFactory.createMethodFromText(method.toString(), mClass));
    }
  }
}
```

在写入类字段的时候，需要判断是否需要添加前缀，在`Constant`中列举了一些需要特殊处理的对象

``` java
    protected void generateFields() {
        for (ElementBean element : mElements) {
            if (!element.used) {
                continue;
            }

            StringBuilder injection = new StringBuilder();
            injection.append("@BindView");
            injection.append('(');
            injection.append(element.getGenerateValue(generateId));
            injection.append(")");
            if (element.nameFull != null && element.nameFull.length() > 0) { // custom package+class
                injection.append(element.nameFull);
            } else if (Constant.paths.containsKey(element.name)) { // listed class
                injection.append(Constant.paths.get(element.name));
            } else { // android.widget
                injection.append("android.widget.");
                injection.append(element.name);
            }
            injection.append(" ");
            injection.append(element.fieldName);
            injection.append(";");
            mClass.add(mFactory.createFieldFromText(injection.toString(), mClass));
        }
    }
```

写入完成后格式化一下代买，要不然写入的字段会是这样：`android.widget.TextView userNameTextView`

``` java
JavaCodeStyleManager styleManager = JavaCodeStyleManager.getInstance(mProject);
styleManager.optimizeImports(mFile);
styleManager.shortenClassReferences(mClass);
new ReformatCodeProcessor(mProject, mClass.getContainingFile(), null, false).runWithoutProgress();
```

到此为止，就已经完成了我们的工作。



----

以上