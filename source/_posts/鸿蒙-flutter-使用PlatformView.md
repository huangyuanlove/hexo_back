---
title: 鸿蒙-flutter-使用PlatformView
tags: [HarmonyOS，Flutter]
date: 2025-06-10 16:14:54
keywords:
---

我们自己的业务比较简单，基本上没有使用PlatformView，所有的页面要么是原生，要么是flutter，没有这种在flutter页面上展示原生控件的需求。
这里介绍一下如何在纯flutter项目中使用platformView展示鸿蒙组件。

## 准备
按照之前的环境搭建和第一个helloworld，搭建好环境，运行起来。

## 原生侧
使用DevEco打开项目工程下的ohos文件夹，DevEco会将该文件夹识别为一个鸿蒙项目，可以获得完整的代码提示和语法高亮。
我们先从底层向接口方向编写代码。

### 需要展示的View
定义一个用来在Flutter上展示的 Component。

``` TypeScript
@Component
struct ButtonComponent {
    @Prop params: Params
    customView: CustomView = this.params.platformView as CustomView
    @StorageLink('numValue') storageLink: string = "first"
    @State bkColor: Color = Color.Red

    build() {
      Column() {
        Button("发送数据给Flutter")
          .border({ width: 2, color: Color.Blue})
          .backgroundColor(this.bkColor)
          .onTouch((event: TouchEvent) => {
            console.log("nodeController button on touched")
          })
          .onClick((event: ClickEvent) => {
            this.customView.sendMessage();
            console.log("nodeController button on click")
          })

        Text(`来自Flutter的数据 : ${this.storageLink}`)
          .onTouch((event: TouchEvent) => {
            console.log("nodeController text on touched")
          })

        }.alignItems(HorizontalAlign.Center)
        .justifyContent(FlexAlign.Center)
        .direction(Direction.Ltr)
        .width('100%')
        .height('100%')
    }
}
```

### 