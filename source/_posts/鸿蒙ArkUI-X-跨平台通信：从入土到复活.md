---
title: 鸿蒙ArkUI-X 跨平台通信：从入土到复活
tags: [HarmonyOS]
date: 2024-03-27 14:21:02
keywords: HarmonyOS,鸿蒙应用开发,鸿蒙手机应用开发,ArkUI-X,鸿蒙跨平台,ArkUI跨平台
---

----
2024.01.31 更新  
在上一篇 [鸿蒙跨平台 ArkUI-X从入门到入土 ](https://juejin.cn/post/7327910163628294154)中提到创建 Bridge 对象时失败的问题，在本文中提到的问题又重新验证了几次，咨询了一下相关人员，结论是这样的
1. 在 Arkui-X 中，如果 Bridge 对象声明为成员变量并且立即创建，这时候 preview 会白屏，是加载界面时就挂了，因为这个bridge对象，是需要 native 侧的文件支持的，比如Android中的libbridge.so(集成产物到 Android 工程时复制过去的)。这时候集成到 Android 工程中是正常运行的。
2. DevEco 中的 preview 相当于纯鸿蒙系统(HarmonyOS next),在纯鸿蒙系统中是无法使用Bridge 的，因为纯鸿蒙上没有这个 so 库。所以在创建这个 Bridge 对象的时候需要判断一下是不是跨平台，一般从deviceInfo.osFullName判断：  
`let osName: string = deviceInfo.osFullName;`获取对应OS名字，该接口已支持跨平台，不同平台上其返回值如下:

- OpenHarmony上，osName等于`OpenHarmony XXX`
- Android上，osName等于`Android XXX`
- iOS上，osName等于`iOS XXX`  
具体文档在这里 [平台差异化](https://gitee.com/arkui-x/docs/blob/master/zh-cn/application-dev/quick-start/platform-different-introduction.md#%E5%B9%B3%E5%8F%B0%E5%B7%AE%E5%BC%82%E5%8C%96)
----

### 前言
话说前两天刚调研了 ArkUI-X 跨平台方案，最终卡死在了跨平台和 native 通信上，文章在这里[鸿蒙跨平台 ArkUI-X从入门到入土](https://juejin.cn/post/7327910163628294154)，今天在社区的帮助下跑通了通信方案，该挖出来复活了。  
**注意文章所说的官方是指社区，并不是指华为公司，更不是其他**

### 准备

这里只对 Android 侧进行了实现，iOS 侧因为没有实体机的原因，先放一放，原理都一样，代码也差不多。官方文档先放在这里了 [平台桥接开发指南](https://gitee.com/arkui-x/docs/blob/master/zh-cn/application-dev/tutorial/how-to-use-bridge-on-android.md#%E5%B9%B3%E5%8F%B0%E6%A1%A5%E6%8E%A5%E5%BC%80%E5%8F%91%E6%8C%87%E5%8D%97)

> 平台桥接用于客户端（ArkUI）和平台（Android或iOS）之间传递消息，即用于ArkUI与平台双向数据传递、ArkUI侧调用平台的方法、平台调用ArkUI侧的方法。本文主要介绍Android平台与ArkUI交互，ArkUI侧具体用法请参考[Bridge API](https://gitee.com/arkui-x/docs/blob/master/zh-cn/application-dev/reference/apis/js-apis-bridge.md)，Android侧参考[BridgePlugin](https://gitee.com/arkui-x/docs/blob/master/zh-cn/application-dev/reference/arkui-for-android/BridgePlugin.md)。

官方在 Android 侧提供了一个抽象类`BridgePlugin`，我们需要继承它实现一些方法来进行通信。在 ArkUI-X 侧同样提供了`'@arkui-x.bridge`包来进行通信。


#### ArkUI-X 侧 Bridge

先看下ArkUI-X 侧提供的方案，官方文档在这里 [@arkui-x.bridge.d.ts (平台桥接)](https://gitee.com/arkui-x/docs/blob/master/zh-cn/application-dev/reference/apis/js-apis-bridge.md#arkui-xbridgedts-%E5%B9%B3%E5%8F%B0%E6%A1%A5%E6%8E%A5)。  
在官方提供的[场景示例中](https://gitee.com/arkui-x/docs/blob/master/zh-cn/application-dev/tutorial/how-to-use-bridge-on-android.md#%E5%9C%BA%E6%99%AF%E7%A4%BA%E4%BE%8B)中，是在页面(也就是被`@Entry`装饰的类)中创建的，但是在实践中发现不能正常运行，会创建 Bridge 对象时会报错

> Error message: Cannot read property createBridge of undefined

指向了`private bridgeImpl = bridge.createBridge('Bridge');`这一行代码，向官方提交了 issue，在其帮助下，将创建 Bridge 对象的代码放在了另外的 ets 文件中可以正常运行。
##### 创建 Bridge
先导包 `import Bridge from '@arkui-x.bridge';`  
再创建` bridgeObj: BridgeObject = Bridge.createBridge('Bridge');`  
需要注意的是，这里传入的参数值需要**和 native 侧一致，否则无法调用**。 


#### Android 侧 BridgePlugin
看先 Android 侧提供的方案，官方文档在这里 [BridgePlugin (平台桥接)](https://gitee.com/arkui-x/docs/blob/master/zh-cn/application-dev/reference/arkui-for-android/BridgePlugin.md#bridgeplugin-%E5%B9%B3%E5%8F%B0%E6%A1%A5%E6%8E%A5) 。  
暴露的 api 也不多，包括构造方法、callMethod、sendMessage，两个回调监听：setMessageListener和setMethodResultListener。

##### 创建 BridgePlugin
一般来讲，我们会自己写个类继承`BridgePlugin`来进行操作
``` java
public class ArkUIBridge extends BridgePlugin{
    private final String TAG = "ArkUIBridge";
    public ArkUIBridge(Context context, String bridgeName, int instanceId) {
        super(context, bridgeName, instanceId);
    }
}
```
注意这里的`bridgeName`参数，传入的值**必须**与 ArkUI-X 侧一致，至于 instanceId 则是`StageActivity`这个用来展示 ArkUI-X 内容的容器提供的方法，其实也就是调用的`InstanceIdGenerator.getAndIncrement()`，具体实现如下
``` Java
package ohos.stage.ability.adapter;

import java.util.concurrent.atomic.AtomicInteger;

public final class InstanceIdGenerator {
    private static final AtomicInteger ID_GENERATOR = new AtomicInteger(1);

    public InstanceIdGenerator() {
    }

    public static int getAndIncrement() {
        return ID_GENERATOR.getAndIncrement();
    }

    public static int get() {
        return ID_GENERATOR.get();
    }
}
```
我们可以在其他位置调用`InstanceIdGenerator.get()`来获取到 id。但需要注意，每次创建 ArkUI-X 产物的容器页面也就是StageActivity时，该 id 都会自增，如果 id 无法对应则无法互相通信

#### ArkUI侧向Android侧传递数据 

``` TypeScript
// xxx.ets
bridgeImpl.sendMessage('text').then((res)=>{
    // 监听Android侧的回执
    console.log('response: ' + res);
}).catch((err) => {
    console.log('error: ' + JSON.stringify(err));
});
```
在 Android 侧接收消息,在构造方法里面设置一下监听事件
``` Java
//ArkUIBridge extends BridgePlugin
public ArkUIBridge(Context context, String bridgeName, int instanceId) {
    super(context, bridgeName, instanceId);
    setMessageListener(new IMessageListener() {
        @Override
        public Object onMessage(Object o) {
            Log.e(TAG,"onMessage-->" + o.toString());
            JSONObject result = new JSONObject();
            try {
                result.put("platform","Android");
                result.put("result_code",0);
            }catch (Exception e){
                e.printStackTrace();
            }
            return result.toString();
        }
        @Override
        public void onMessageResponse(Object o) {
            Log.e(TAG,"onMessageResponse-->" + o.toString());
        }
    });
}

```
#### Android侧向ArkUI-X侧传递数据 
方式都一样，需要在 ArkUI-X 侧设置一下监听事件
``` TypeScript
this.bridge = Bridge.createBridge('BridgeCommon');
this.bridge!.setMessageListener((message: string) => {
if (message) {
  console.log(`receive message：${message}`);
  this.scanResult = message
}
return "ArkUI-X setMessageListener";
})
```
在 Android 侧，
```Java
////ArkUIBridge extends BridgePlugin
public void sendMessageToArkUI(){
    JSONObject jsonObject = new JSONObject();
    try {
        jsonObject.put("code","0");
        jsonObject.put("msg","扫描结果");
        jsonObject.put("data","scan result from Android");

    }catch (Exception e){
        e.printStackTrace();
    }
    Log.e(TAG,"toScan before sendMessage" );
    sendMessage(jsonObject.toString());

}
```
需要注意的是，在 Android 中调用 sendMessage 方法是没有返回值的，ArkUI-X 侧收到消息后的返回值是在`setMessageListener`的`onMessageResponse`回调中接收的。

#### ArkUI-X 侧调用 Android 侧的方法
在 ArkUI-X 中

``` TypeScript
  async getAppVersion(): Promise<string> {
    this.initBridge();//创建 bridge 对象
    let params:Record<string,Bridge.Parameter> ={
      "name":"xuan",
      "age":18
    }
    let result = await this.bridge!.callMethod('getAppVersion',params);
    console.log('getAppVersion返回值：' + result)
    return result!.toString();
  }

```
在 Android
``` Java
public String getAppVersion(JSONObject params){
    Log.e(TAG,"getAppVersion from arkui-x，params--> "  );
    if(params == null){
        Log.e(TAG,"is null");
    }else{
        Log.e(TAG,params.toString());
    }
    JSONObject jsonObject = new JSONObject();
    try {
        jsonObject.put("version",BuildConfig.VERSION_NAME);
        jsonObject.put("buildVersion","getAppVersion(Object params)");

    }catch (Exception e){
        e.printStackTrace();
    }

    return jsonObject.toString();
}
```
需要注意的是，两侧都不支持方法重载，在 Android 侧是通过 HashMap 保存的在 BridgePlugin 中的方法并且是以方法名为 key，java.lang.reflect.Method为值。在 Android 侧的方法会被自动注册，不需要我们调用代码注册。

####  Android 侧调用 ArkUI-X 侧 的方法
在 ArkUI-X 中，需要自己调用registerMethod方法来注册供 native 调用的方法。
``` TypeScript
//方法声明
getString(parameters?: Record<string,Bridge.Message>):Bridge.ResultValue {
    console.log(`----调用 getString：parameters-->${JSON.stringify(parameters)}`);
return 'call js getString success';
}
//注册方法
this.bridge!.registerMethod({name:"getString",method:this.getString})
```
在 Android 侧
``` Java
JSONObject params = new JSONObject();
try {
    params.put("name","xuan");
    params.put("age",18);
}catch (Exception e){
    e.printStackTrace();
}
Object[] paramObject = {params};
MethodData methodData = new MethodData("getString", paramObject);
callMethod(methodData);
```
同样的，Android 调用 ArkUI 的方法并没有返回值，需要在`setMethodResultListener`的`onSuccess`方法中获取
``` Java
//设置调用 ArkUI-X 方法的结果回调
setMethodResultListener(new IMethodResult() {
    @Override
    public void onSuccess(Object o) {
        Log.e(TAG,"IMethodResult#onSuccess-->" +o.toString());
    }

    @Override
    public void onError(String s, int i, String s1) {

    }

    @Override
    public void onMethodCancel(String s) {

    }
});
```


### 注意事项

#### BridgePlugin 中提供给 ArkUI-X 调用的方法不支持方法重载
原因上面也说了，是因为保存的时候是用方法名作为 key 保存在 HashMap 中的，重载也没用，虽然写了不报错，但结果不保证。也看一下为啥 Android 不用自己写代码注册供 ArkUI-X调用的方法。  
在`ohos.ace.adapter.capability.bridge.BridgePlugin`这个类中，重点关注`HashMap<String, Method> methodsMap_`这个成员变量和这几个方法：  
* protected Object jsCallMethod(Object object, MethodData methodData) 
* private Method findMethod(String methodName)
* private void registerMethod(String methodName, Method methods)  

当 ArkUI-X 调用 Android 方法时，首先调用的是`jsCallMethod`，在`jsCallMethod`中首先调用`findMethod`方法从`methodsMap_`中获取对应的方法，找了则直接调用。没找到则反射获取 BridgePlugin 实现类中的方法，然后使用方法名做匹配，找到对应的方法。到这里也就解释了为啥不支持方法重载。也解释了为啥方法参数对应不上会有异常。  
cpp 的源码在 [https://gitee.com/arkui-x/arkui_for_android/tree/master/capability/java/jni/bridge](https://gitee.com/arkui-x/arkui_for_android/tree/master/capability/java/jni/bridge)  
Java 源码在 [https://gitee.com/arkui-x/arkui_for_android/tree/master/capability/java/src/ohos/ace/adapter/capability/bridge](https://gitee.com/arkui-x/arkui_for_android/tree/master/capability/java/src/ohos/ace/adapter/capability/bridge)

![image.png](image/HarmonyOS/arkui-x_debug.png)

#### 参数类型对应关系
Arkui-X 中callMethod是这么声明的
``` TypeScript
callMethod(methodName: string, parameters?: Record<string, Parameter>):
callMethod(methodName: string, ...parameters: Array<any>): Promise<ResultValue>;
```
sendMessage是这么声明的
``` TypeScript
sendMessage(message: Message, callback: AsyncCallback<Response>): void;
sendMessage(message: Message): Promise<Response>;
```

``` TypeScript
type S = number | boolean | string | null;
type T = S | Array<number> | Array<boolean> | Array<string>;
type Message = T | Record<string, T>;
type Parameter = Message;
type Response = Message;
type ResultValue = T | Map<string, T>;
```
在 Android 中sendMessage
``` Java
public void sendMessage(Object data){}
```
callMethod
``` Java
public void callMethod(MethodData methodData)
```
而 MethodData 只有两个成员变量
``` Java
public class MethodData {
    private String methodName_;
    private Object[] Parameters_;

    public MethodData(String methodName, Object[] parameter) {
        this.methodName_ = methodName;
        this.Parameters_ = parameter;
    }

    public String getMethodName() {
        return this.methodName_;
    }

    public Object[] getMethodParameter() {
        return this.Parameters_;
    }
}

```

那么在使用的时候可以这样:  
##### ArkUI-X主动调用 Android
在 ArkUI-X 中调用
``` TypeScript
let params:Record<string,Bridge.Parameter> ={
  "name":"xuan",
  "age":18
}
let result = await this.bridge!.callMethod('getAppVersion',params);

```
在 Android 端对应参数类型
``` Java
public String getAppVersion(JSONObject params){
}
```
##### Android 主动调用 ArkUI-X
在 Android 中调用
``` Java
JSONObject params = new JSONObject();
try {
    params.put("name","xuan");
    params.put("age",18);
}catch (Exception e){
    e.printStackTrace();
}
Object[] paramObject = {params};
MethodData methodData = new MethodData("getString", paramObject);
callMethod(methodData);
```
在 ArkUI-X 中对应类型
``` TypeScript
getString(parameters?: Record<string,Bridge.Message>):Bridge.ResultValue {
console.log(`----调用 getString：parameters-->${JSON.stringify(parameters)}`);
return 'call js getString success';
}
```

#### BrigePlugin的bridgeType_
BrigePlugin提供了一个可以指定`bridgeType_`的构造方法，
![image.png](image/HarmonyOS/arkui-x_bridge_type.png) 
如果我们不指定类型的话，默认就是 `BridgeType.JSON_TYPE`，传一些非二进制的数据。但假如我们需要穿一些二进制数据，比如图片、音视频数据等，可以指定为`BridgeType.BINARY_TYPE`。


----
以上





