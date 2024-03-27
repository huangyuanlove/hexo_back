---
title: 鸿蒙跨平台 ArkUI-X从入门到入土
tags: [HarmonyOS]
date: 2024-03-27 11:13:02
keywords: HarmonyOS,鸿蒙应用开发,鸿蒙手机应用开发,ArkUI-X,鸿蒙跨平台,ArkUI跨平台
---


----
2024.01.31 更新  
码完了
[鸿蒙ArkUI-X 跨平台通信：从入土到复活](https://juejin.cn/post/7329310941106421811)  
在本文中提到的问题又重新验证了几次，咨询了一下相关人员，结论是这样的
1. 在 Arkui-X 中，如果 Bridge 对象声明为成员变量并且立即创建，这时候 preview 会白屏，是加载界面时就挂了，因为这个bridge对象，是需要 native 侧的文件支持的，比如Android中的libbridge.so(集成产物到 Android 工程时复制过去的)。这时候集成到 Android 工程中是正常运行的。
2. DevEco 中的 preview 相当于纯鸿蒙系统(HarmonyOS next),在纯鸿蒙系统中是无法使用Bridge 的，因为纯鸿蒙上没有这个 so 库。所以在创建这个 Bridge 对象的时候需要判断一下是不是跨平台，一般从deviceInfo.osFullName判断：  
`let osName: string = deviceInfo.osFullName;`获取对应OS名字，该接口已支持跨平台，不同平台上其返回值如下:

- OpenHarmony上，osName等于`OpenHarmony XXX`
- Android上，osName等于`Android XXX`
- iOS上，osName等于`iOS XXX`  
具体文档在这里 [平台差异化](https://gitee.com/arkui-x/docs/blob/master/zh-cn/application-dev/quick-start/platform-different-introduction.md#%E5%B9%B3%E5%8F%B0%E5%B7%AE%E5%BC%82%E5%8C%96)
----
2024.01.28 更新
~~上面~~文章中提到跨平台通信(Bridge)的问题在社区的帮助下解决了，方案就不要在页面(也就是@Entry修饰)中进行初始化，可以写个工具类，在工具类中初始化，虽然 debug 也是提示 undefined，但运行的时候可以正常工作。  
新的博客记录已经在码了。。。。

----


### 前言
喊了好长时间要做鸿蒙应用，自己也写了一点，但要同时照顾三个移动平台有点恶心，大致看了一下鸿蒙社区的 arkui-x 跨平台方案 [https://gitee.com/arkui-x](https://gitee.com/arkui-x) ,先调研一下试试水
**注意文章所说的官方是指社区，并不是指华为公司，更不是其他**

### 限制
丑话说在前头，先说限制，按照官方文档说法，忘记在哪里看到了
Android系统版本8+ 且仅 arm 设备支持
iOS系统版本 10+ 且仅 arm 设备支持

### 准备
官方文档看这里：[https://gitee.com/arkui-x/docs/blob/master/zh-cn/application-dev/README.md](https://gitee.com/arkui-x/docs/blob/master/zh-cn/application-dev/README.md)

官方仓库在这里：[https://gitee.com/arkui-x](https://gitee.com/arkui-x)

使用官方的套件还是需要申请，方式和之前一样，就是找商务谈合作，签协议。然后给账号开通下载权限然后去下载。

#### IDE
这里我们可以使用 OpenHarmony 社区提供的开发套件

![image.png](image/HarmonyOS/deveco.png)
下载链接：

https://gitee.com/openharmony/docs/blob/master/zh-cn/release-notes/OpenHarmony-v4.0-release.md#%E9%85%8D%E5%A5%97%E5%85%B3%E7%B3%BB

安装步骤都一样，**注意 node 和ohpm版本**，选择ide建议的版本，可以重新下载，也可以使用本机上已经安装好的，我这里用的 node是 16.20.0，不要头铁搞个 18.x.x 20.x.x的版本，可能会有一些诡异的问题

#### 配置

启动 IDE，页面左侧有diagnose可以检测一些基础配置和网络连接情况。点击左侧Customize，在右侧底部点击"All settings"进入首选项配置

![image.png](image/HarmonyOS/deveco_customize.png)

选择 SDK，安装 OpenHarmony SDK api 10，安装完成之后再安装 ArkUI-X

![image.png](image/HarmonyOS/deveco_sdk.png)

安装完成后就可以了，没有其他需要安装的了

### 开始

#### 创建工程

没有提供直接创建跨平台应用的地方，目前只能用 import 的形式

![image.png](image/HarmonyOS/deveco_import_project.png)

找到 import Sample,新页面左上角选择 OpenHarmony，下面会出现 ArkUI-X，这里用 HelloWorld 来做示例。

![image.png](image/HarmonyOS/select_sample_to_improt.png)

打开工程后开始自动同步，但这里会失败，因为各种插件版本不适配，点一下蓝色的文字，会帮你全部修改好，重新同步，到这里就已经全部准备好了。

![image.png](image/HarmonyOS/sync_failed.png)

#### 编译

窗口顶部菜单 Build--> Build Hap(s)/APP(s)-->Build APP(s)。会同时构建 Android 和 iOS 产物

![image.png](image/HarmonyOS/build_result.png)

不出意外的话出意外了，打包失败，但这时对应的资源都已经复制到相应的文件夹中了。对应的说明可以看这里

https://gitee.com/arkui-x/docs/blob/master/zh-cn/application-dev/quick-start/package-structure-guide.md

这时候我们进入到项目工程的 .arkui-x/android目录下，执行 **./gradlew assembleDebug** 来编译 android 安装包。注意这里 gradlew 可能没有执行权限，**chmod +x gradlew** 给一下执行权限就好了。

对于 iOS 工程，可以用 Xcode 打开之后配置一下签名然后打包。

到这里，新建工程编译多平台就已经完成了。但我们有很多项目不是从头开始，部分新增内容需要使用 add-on方式，这里以 Android 为例

### 添加到现有工程

接上面 **窗口顶部菜单 Build--> Build Hap(s)/APP(s)-->Build APP(s)。会同时构建 Android 和 iOS 产物** 之后，对应资源文件已经复制到对相应文件夹了。

先准备一个 Android 项目，注意一个 ArkUI-x 跨平台的版本要求，Android 系统 8 以上，只支持 arm 设备。

看一下 .arkui-x/android的代码，就一个**继承自StageApplication的MyApplication**和一个继承自 **Activity 的EntryEntryAbilityActivity，** 该类名通过通过module名和ability名拼接而得，一个ability对应一个Android工程侧的Activity类。

#### 集成
1. libs 下面的 jar 包和so 文件复制到 Android 工程中，注意arkui-x 的 Android 工程中指定了存放 so 文件的文件夹就是 libs，复制到 Android 工程中的时候别整错了
2. assets文件夹下的文件也原封不动的复制到 Android 工程的 assets 文件夹中
3. Android 工程中的 Application改造，这里提供了三种方式  
  3.1 继承StageApplication

```Java
import ohos.stage.ability.adapter.StageApplication;

public class HiStageApplication extends StageApplication {
    @Override
    public void onCreate() {
        super.onCreate();
    }
}
```
    
   3.2 在 Application 中使用StageApplicationDelegate

这个方法和 StageApplication 源码一样

``` java
public class MyApplication extends Application {
    
    private StageApplicationDelegate appDelegate = null;
    public StageApplication() {
    }


    public void onCreate() {
        Log.i("StageApplication", "StageApplication onCreate called");
        super.onCreate();
        this.appDelegate = new StageApplicationDelegate();
        this.appDelegate.initApplication(this);
    }


    public void onConfigurationChanged(Configuration newConfig) {
        Log.i("StageApplication", "StageApplication onConfigurationChanged called");
        super.onConfigurationChanged(newConfig);
        if (this.appDelegate == null) {
            Log.e("StageApplication", "appDelegate is null");
        } else {
            this.appDelegate.onConfigurationChanged(newConfig);
        }
    }
}
```

3.3 在 Activity 中使用 StageApplicationDelegat

``` java
import android.app.Activity;
import ohos.stage.ability.adapter.StageApplicationDelegate;


public class EntryEntryAbilityActivity extends Activity {


    private StageApplicationDelegate appDelegate_ = null;


    @Override
    protected void onCreate(Bundle savedInstanceState) {
        appDelegate_ = new StageApplicationDelegate();
        appDelegate_.initApplication(this.getApplication());
        super.onCreate(savedInstanceState);
    }
}
```


4. 用于展示内容的 Activity

直接复制也行，自己创建一个同名 Activity 把内容复制过来，清单文件中注册一下也行。

1.  原生拉起 arkui-x 跨平台页面并传参

使用原生Activity拉起Ability时，需使用原生应用的startActivity方法，参数的传递需要通过Intent中的putExtra()进行设置，规则如下：

key值为params

value为json格式

``` java
Intent intent = new Intent();
        intent.setClass(this, EntryEntryAbilityTwoActivity.class);
        intent.putExtra("params",
                "{"params":[{"key":"bool","type":1,"value":"true"}," +
                "{"key":"double","type":9,"value":"2.3"}," +
                "{"key":"int","type":5,"value":"2"}," +
                "{"key":"string","type":10,"value":"test"}]}");
        startActivity(intent);
```


至此，集成完成。
在 Android 项目中调用一下就可以看到页面了

### arkui-x 和 native 通信

原生和跨平台通信是非常重要的一个功能，也是不可或缺的一部分，官方给出了桥接平台 Bridge，在 Android、iOS 和 arkui-x 侧都有配套说明：
>平台桥接用于客户端（ArkUI）和平台（Android或iOS）之间传递消息，即用于ArkUI与平台双向数据传递、ArkUI侧调用平台的方法、平台调用ArkUI侧的方法。本文主要介绍Android平台与ArkUI交互，ArkUI侧具体用法请参考Bridge API，Android侧参考BridgePlugin。  

这里也给出了一个
[场景示例](https://gitee.com/arkui-x/docs/blob/master/zh-cn/application-dev/tutorial/how-to-use-bridge-on-android.md#%E5%9C%BA%E6%99%AF%E7%A4%BA%E4%BE%8B)，但奇怪的是不能正常运行，复现步骤和现象放在这个 [issue](https://gitee.com/arkui-x/docs/issues/I8YWM2?from=project-issue) 中了。目前是待确认状态。

到这里也没有需要继续下去的东西，就先入土吧，上面这个问题有答案了再挖出来继续。

----
以上

