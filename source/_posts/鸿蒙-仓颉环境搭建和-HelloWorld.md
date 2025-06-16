---
title: 鸿蒙-仓颉环境搭建和 HelloWorld
tags: [HarmonyOS,仓颉]
date: 2025-06-13 11:13:16
keywords: HarmonyOS,仓颉,仓颉开发鸿蒙
---
虽然HarmonyOS NEXT Cangjie正式版本测试活动开始小范围招募了，报名链接还不太想让大范围转发，这里就不放了。
![仓颉-鸿蒙正式版招募](image/cangjie/仓颉-鸿蒙正式版招募.png)
![HarmonyOS_NEXT_Cangjie正式版本测试活动.png](image/cangjie/HarmonyOS_NEXT_Cangjie正式版本测试活动.png)
但公测版还是申请审核制度。我们可以在仓颉官网上找到[公测版报名链接](https://developer.huawei.com/consumer/cn/activityDetail/cangjie-beta/),审核挺快的，一两天就通过了。
但最近他们可能在忙 hdc 活动，审核可能会慢些。

## 下载
官方下载页面：https://developer.huawei.com/consumer/cn/download/
注意：公测版申请不通过的话，是看到不插件的。
![Harmony_next_cangjie_beta](image/cangjie/Harmony_next_cangjie_beta.png)  

建议使用最新稳定版本的DevEco，目前是 `DevEco Studio 5.1.0 Release`,版本号是`Build Version
5.1.0.828`。    
最新的插件版本为：`DevEco Studio-Cangjie Plugin 5.0.13.200 Canary`

## 安装
先安装 DevEco，安装完成后我们打开软件，点击左侧的`Customize`,右侧最下方点击`All settings`
![找到所有设置](image/cangjie/找到所有设置.png) 
在新的窗口左侧找到`Plugins`,右侧点击齿轮，在弹出的菜单中选择`Install from Disk`
![安装仓颉插件](image/cangjie/安装仓颉插件.png)  
选择我们刚才下载的插件压缩包，安装完成后需要重启一下DevEco

## 提示
安装仓颉插件后，需要重新启动 DevEco Studio。初始化工程时，会自动配置仓颉 SDK，仓颉 SDK 存放的路径在 macOS 系统下默认为 $HOME/.cangjie-sdk，Windows 下默认为 %USERPROFILE%/.cangjie-sdk。如需指定 .cangjie-sdk 的存放路径，请在安装插件前配置系统环境变量，变量名为 DEVECO_CANGJIE_PATH，变量值为要存放的路径。
配置系统环境变量后，请重启 DevEco Studio，使环境变量生效。

## HelloWorld
现在我们打开 DevEco，创建一个新的工程，选择\[Cangjie\]Empty Ability。输入项目名称、应用包名等信息后点击 finish 就可以了。有一点不同的是，目前 cangjie 只支持 phone 这个选项。
![create_cangjie_harmony_project](image/cangjie/create_cangjie_harmony_project.png)。
等待项目同步完成后，我们就可以进行开发了。

## 工程目录

![鸿蒙仓颉目录](image/cangjie/鸿蒙仓颉目录.png)  
我们可以看到鸿蒙仓颉目录和 ArkTS 工程的目录几乎是相同的，降低了我们上手开发的难度。

## 编译构建
和使用`ArkTS`一样，我们同样需要对应用进行签名，之后才可以编译运行。
这里需要注意的是仓颉工程默认编译架构为arm64-v8a，因此在使用x86模拟器时,需要编译出x86_64版本的so。
我们需要在仓颉模块的`build-profile.json5`配置文件中，为`cangjieOptions.abiFilters`的值增加**x86_64**，具体编译配置如下：
``` json
"buildOption": {      // 配置项目在构建过程中使用的相关配置
  "cangjieOptions": { // 仓颉相关配置
    "path": "./src/main/cangjie/cjpm.toml", // cjpm配置文件路径，提供仓颉构建配置
    "abiFilters": ["arm64-v8a", "x86_64"]   // 自定义仓颉编译架构，默认编译架构为arm64-v8a
  }
}
```
之后我们就可以编译运行了。
![仓颉鸿蒙应用](image/cangjie/harmony_cangjie_demo.png)

## 功能开发
我们做个最简单的页面跳转看看
先创建一个新的文件:`second_page.cj`,然后简单写一下页面布局
``` typescript
package ohos_app_cangjie_entry

internal import ohos.base.LengthProp
internal import ohos.component.Column
internal import ohos.component.Row
internal import ohos.component.Button
internal import ohos.component.Text
internal import ohos.component.CustomView
internal import ohos.component.CJEntry
internal import ohos.component.loadNativeView
internal import ohos.state_manage.SubscriberManager
internal import ohos.state_manage.ObservedProperty
internal import ohos.state_manage.LocalStorage
import ohos.state_macro_manage.Entry
import ohos.state_macro_manage.Component
import ohos.state_macro_manage.State
import ohos.state_macro_manage.r
import ohos.hiappevent.Event
import ohos.router.Router
import ohos.component.Alignment

@Entry
@Component
class SecondPage {
    @State
    var message: String = "Second Page"
    func build() {
        Column {
            Button(message).onClick {
                evt => AppLog.info("Second Page")
            }.fontSize(40).height(80)

            Button("返回上个页面").onClick({
                event => Router.back()
            })
        }.width(100.percent).align(Alignment.Start)
    }
}


```
然后我们在第一个页面`index.cj`中添加一下跳转

``` typescript
package ohos_app_cangjie_entry

internal import ohos.base.LengthProp
internal import ohos.component.Column
internal import ohos.component.Row
internal import ohos.component.Button
internal import ohos.component.Text
internal import ohos.component.CustomView
internal import ohos.component.CJEntry
internal import ohos.component.loadNativeView
internal import ohos.state_manage.SubscriberManager
internal import ohos.state_manage.ObservedProperty
internal import ohos.state_manage.LocalStorage
import ohos.state_macro_manage.Entry
import ohos.state_macro_manage.Component
import ohos.state_macro_manage.State
import ohos.state_macro_manage.r
import ohos.hiappevent.Event
import ohos.router.Router
import ohos.component.Alignment

@Entry
@Component
class EntryView {
    @State
    var message: String = "Hello Cangjie"
    func build() {
        Column {
            Button(message).onClick {
                evt => AppLog.info("Hello Cangjie")
            }.fontSize(40).height(80)

            Button("跳转到下个页面").onClick(
                {
                    event =>
                    AppLog.debug("点击了跳转到下个页面")
                    Router.push(url: "SecondPage", params: "123456789")
                }
            )
        }.width(100.percent).align(Alignment.Start)
    }
}

```
我们看下效果：
![页面跳转](image/cangjie/cangjie_harmony_demo_router.gif)


## 其他
说点题外话：
一开始我还很纳闷，应用启动的时候是如何识别第一个需要加载的页面的。 直到我看完教程，发现使用 Router 跳转时是直接写死的目标页面的`class name`，回去看了一下`MainAbility`代码，发现是在`onWindowStageCreate`这个方法中调用了`windowStage.loadContent("EntryView")`，直接加载了`index.cj`中的`class name`。

## 参考资料
官网上貌似没有直达的链接：

[快速入门](https://developer.huawei.com/consumer/cn/doc/cangjie-guides-V5/cj-first-cangjie-app-V5)  
[仓颉 API](https://developer.huawei.com/consumer/cn/doc/cangjie-references-V5/_u4ed3_u9889api-V5)  
[仓颉组件](https://developer.huawei.com/consumer/cn/doc/cangjie-references-V5/_u4ed3_u9889_u7ec4_u4ef6-V5)  

仓颉语言更新或者鸿蒙版本更新时，上面的链接可能会失效，我们可以在[gitcode Cangjie](https://gitcode.com/Cangjie)项目中的鸿蒙版本介绍中获取最新的链接。

## 吐槽
大部分调用的 api 看不到源码，写起来就很难受，有的是因为 api 不熟悉，不知道该传什么类型的参数。
有的是报错找不到对应的解决方案
有时候报错、页面不是预期效果、运行起来和官网示例效果不一样等等，有可能不是我们的问题，
现在有好多 issue 没有关闭，还在优化中，
[https://gitcode.com/Cangjie/CangjieMoBileUsersForm](https://gitcode.com/Cangjie/CangjieMoBileUsersForm/issues)，同样的，访问这个页面也需要权限。。。