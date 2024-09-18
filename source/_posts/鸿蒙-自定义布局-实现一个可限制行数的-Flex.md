---
title: 鸿蒙-自定义布局-实现一个可限制行数的 Flex
tags: [HarmonyOS]
date: 2024-09-18 15:59:02
keywords: HarmonyOS,鸿蒙应用开发,鸿蒙手机应用开发,TextInput,验证码输入框
---
千呼万唤始出来的自动以布局功能终于可以用了，这就给了我们更多自由发挥创造的空间，不再局限于使用已有控件做组合。当然，用 NAPI 和 C|C++页可以实现自己绘制所有内容，更别提还有类似`XComponent`这种东西了。但假如我们只是需要简单的自己控制子控件所在的位置，不需要接管绘制等逻辑，比如实现一个扇形菜单、实现一个可以控制行数的标签列表等，怎么搞嘞？现在鸿蒙提供了`onPlaceChildren`和`onMeasureSize`这两个回调方法，使得我们可以按照自己的意愿来摆放控件。
<!--more-->

## 前提
我们先来了解一下这两个回调方法：
``` TypeScript
onMeasureSize(selfLayoutInfo: GeometryInfo, children: Array<Measurable>, constraint: ConstraintSizeOptions) {}
onPlaceChildren(selfLayoutInfo: GeometryInfo, children: Array<Layoutable>, constraint: ConstraintSizeOptions) {}
```
> ArkUI框架会在自定义组件确定尺寸时，将该自定义组件的节点信息和尺寸范围通过onMeasureSize传递给该开发者。不允许在onMeasureSize函数中改变状态变量。