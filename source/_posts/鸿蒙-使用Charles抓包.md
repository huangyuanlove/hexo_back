---
title: 鸿蒙-使用Charles抓包
tags: [HarmonyOS]
date: 2025-04-20 10:04:25
keywords: HarmonyOS,鸿蒙应用开发,抓包,charles,忽略证书,跳过证书验证
---

## 前言

抓包，对于各位开发者应该不陌生，各种抓包工具应该的都听说过，像 charles、fiddler、Wireshark‌等。在 Android 和 iOS 上抓包都挺简单的，把证书存放到手机上，然后安装一下，网络设置里面配置一下代理，代码里面忽略一下证书校验或者信任一下用户证书就好了。  
但在鸿蒙手机上，似乎第一步把证书存放到手机上就卡住了一部分人。

## 鸿蒙应用中的网络请求

在开发文档中有提到两种网络请求的方法，一开始是用 http，再后来推荐使用 rcp。现在上架的应用估计大部分是用的 http 或者axios 这个封装好的框架进行的网络请求。  

### rcp 抓包

在官方文档中，并没有找到http 如何忽略证书校验或者信任用户证书，只翻到了如何使用自定义证书。  
嘿嘿，问题不大，因为我们用的是 rcp 做的网络请求，自己封装了一下。并且在官方文档中找到了跳过证书校验的配置：
> SecurityConfiguration接口允许开发人员在会话中配置与安全相关的设置，包括证书和服务器身份验证。

其中有个属性：remoteValidation，解释说明是证书颁发机构（CA），用于验证远程服务器的身份。默认值为'system'。  
我们可以配置的类型有：`"system"`、`"skip"`、`CertificateAuthority`、`ValidationCallback`，其中默认值为'system'。
如果未设置此字段，系统CA将被用于验证远程服务器的标识。  
'system'：表示使用系统CA配置。  
'skip'：跳过验证。  
CertificateAuthority：证书颁发机构（CA）验证。  
ValidationCallback：自定义证书校验。  

这不就简单了么，整个 demo 试一下
``` TypeScript
        Button('charles抓包 rcp').onClick((_)=>{
          const session = rcp.createSession();
          const request = new rcp.Request('https:/xxxxxx','GET');
          request.configuration = {
            security: {
              remoteValidation: 'skip',
            },
          };
          session.fetch(request).then((rep: rcp.Response) => {
            console.info(`Response succeeded: ${rep}`);
          }).catch((err: BusinessError) => {
            console.error(`Response err: Code is ${err.code}, message is ${err.message}`);
          });
        });
```

打开抓包软件，手机 wifi 设置里面配置一下代理，就可以看到能抓包了，甚至不需要安装证书。

### http 抓包

由于没有找到如何忽略证书，就和 Android 抓包一样，先把证书安装到手机上。
在抓包软件中导出证书，注意查看一下**证书的有效期**，当然安装抓包软件的电脑上也需要安装一下证书，并且需要信任才行。  
然后使用`hdc file send`将证书发送到手机上，问题就在这里，不知道手机的文件夹目录是啥。
``` shell
hdc file send charles-ssl-proxying-certificate.pem /storage/media/100/local/files/Docs/Download/charles.pem
```
这里的目标路径为`/storage/media/100/local/files/Docs/Download/`,也就是我们在手机文件管理里面看到的`Download`文件夹。这里需要注意的是，需要在后面加目标文件的名字,这也是和Android的`adb psuh`最大的区别，adb 只需要指定到文件夹就好，相当于把文件复制到这个文件夹中，复制之后的名字可以不指定。  
我们可以在 DevEco 的右下角`Device File Browser`把文件夹展开看一下：  
![](image/HarmonyOS/harmony_os_next_file_system.png)

然后我们打开证书安装页面：`hdc shell aa start -a MainAbility -b com.ohos.certmanager `,或者在手机设置-->隐私和安全-->高级-->证书与凭据-->从存储设备安装,点击 CA 证书，会弹出警告弹窗，我们点击继续，找到我们刚才发送到设备的证书，完成安装  

![](image/HarmonyOS/install_pem_tip.png)

随后撸一坨代码测试一下  

``` TypeScript
        Button('charles抓包 http').onClick((_)=>{
          let httpRequest = http.createHttp();
          httpRequest.request(
            // 填写HTTP请求的URL地址，可以带参数也可以不带参数。URL地址需要开发者自定义。请求的参数可以在extraData中指定
            "https://biztest.chunyutianxia.com/user_operation/app_interface/home_page/?app=0&platform=android&systemVer=10&version=10.6.12&app_ver=Build+10.6.12.250402&cyudId=53f38352-da64-4dac-b4e0-1b0cc681f6a0&secureId=e9ddd1fd-fffe-8a9f-57f7-defffdca8058&installId=1742785027244&phoneType=COL-AL10_by_HUAWEI&vendor=chunyu&screen_height=2060&screen_width=1080",
            {
              method: http.RequestMethod.GET, // 可选，默认为http.RequestMethod.GET
              // 开发者根据自身业务需要添加header字段
              header: {
                'Content-Type': 'application/x-www-form-urlencoded'
              },

              expectDataType: http.HttpDataType.STRING, // 可选，指定返回数据的类型
              priority: 1, // 可选，默认为1
              connectTimeout: 60000, // 可选，默认为60000ms
              readTimeout: 60000, // 可选，默认为60000ms
              usingProxy: true, // 可选，默认不使用网络代理，自API 10开始支持该属性
              // caPath:filePath

            }, (err: BusinessError, data: http.HttpResponse) => {
            if (!err) {
              // data.result为HTTP响应内容，可根据业务需要进行解析
              console.info('Result:' + JSON.stringify(data.result));
              console.info('code:' + JSON.stringify(data.responseCode));
              // data.header为HTTP响应头，可根据业务需要进行解析
              console.info('header:' + JSON.stringify(data.header));
              console.info('cookies:' + JSON.stringify(data.cookies)); // 8+
              // 当该请求使用完毕时，调用destroy方法主动销毁
              httpRequest.destroy();
            } else {
              console.error('error:' + JSON.stringify(err));

              // 当该请求使用完毕时，调用destroy方法主动销毁
              httpRequest.destroy();
              console.error("flutter 鸿蒙打点：")
            }
          }
          );

        })
```
同样的操作，熟悉的配方就可以看到抓包结果了。

## 以下是排查过程，没啥参考价值

### 发送文件
一开始使用 hdc 发送文件一直失败，一个原因是找不到正确的文件夹，另外一个原因就是没有加指定目标文件的文件名，路径只写到了某个文件夹。
还想尝试使用手机上登录微信，通过微信发送。
蓝牙配对一下，使用蓝牙发送。
电脑上搞个 ftp，手机上访问下载一下。  
这些方案应该都能解决文件传输问题，但我就想用 hdc 搞定一下，折腾了好半天，搜了一摞一摞的教程。。。

### http 抓包报错

安装完证书，配置好代理之后抓包时发现 http 请求失败，报错`2300060 远程服务器SSL证书或SSH秘钥不正确`.  
刚开始以为是在 http 请求中需求配置点什么属性，比如`usingProxy`这个属性：可以配置属性值类型`boolean`或者`HttpProxy`。  
首先设置为 true，抓包还是不行。  
设置为`HttpProxy`对象，地址就写电脑的 ip 和抓包软件中设置的对应的端口号，结果还是报错。  

然后就以为需要把证书拷贝到沙箱目录，然后走自定义证书那一套流程，结果还是不行，照样是`2300060`这个错误码。  

之后就去翻官方文档，看到有个提示
![](image/HarmonyOS/http_2300060_error_code.png)

然后看了下证书有效期，果然是证书过期了，但奇怪的是在 Android 上可以抓包。解决方案就是在抓包软件里面重置一下证书，再重新导出一下，安装到手机上就可以了。


----

为啥会关心这两个网络请求抓包：因为有个立项比较早的项目，是由前端主导的，当时还没有 rcp，于是选择了axios。后面又立项了另外一个项目，是客户端主导的，并且这时候官方文档也开始推荐使用 rcp 了。

----
以上