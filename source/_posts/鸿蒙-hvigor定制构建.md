---
title: 鸿蒙-hvigor定制构建
tags: [HarmonyOS]
date: 2025-01-17 23:30:13
keywords: HarmonyOS,鸿蒙应用开发,鸿蒙手机应用开发,构建,hvigor
---

## 前言
之前需要发版时都是在开发机上修改一下相关配置，比如签名文件、三方SDK参数等，然后打包上传到应用商店。略显繁琐，也担心某次打包会有漏改错改的配置。现在使用jenkins搭建了构建流水线，希望可以根据传入的参数不同，替换配置文件中的字段。翻看文档后发现可以在`hvigorfile.ts`中接收部分编译配置。

## BuildProfile
该类和 Android 项目中的 BuildConfig类很像，也是在编译构建时生成的。我们可以通过该类在运行时获取编译构建参数，也可以在`build-profile.json5`中通过buildProfileFields增加自定义字段，从而在运行时获取自定义的参数。

### 实践
项目代码已经迭代了将近10年，有些功能的添加没有办法做到完美向下兼容，只能在请求参数中添加当前应用版本号，服务端根据版本号来判断需要下发哪些数据。但鸿蒙版本是刚开发开发，在一个版本内无法完成全部功能，需要分版本按紧急程度开发，因此版本号也不能直接和 Android、iOS 对齐，也是从 1.0.0 版本开始发版。所以无法在请求参数中直接传递应用版本号。因此我们将当前适配的版本号写入到`BuildProfile.ets`文件中，方便各个业务调用。

我们在项目根目录下的`build-profile.json5`文件中添加如下内容就可以将自定义的字段写入到该文件中.
``` TypeScript
{
	app: {
		products: [{
			name: "default",
			signingConfig: "default",
			compatibleSdkVersion: "5.0.0(12)",
			runtimeOS: "HarmonyOS",
			buildOption: {
				arkOptions: {
					buildProfileFields: {
						online: false,
						version_to_servier: "5.11.10",
					},
				}
			},
		},
		]
	}
}
```
自定义参数可以在`buildOption`、`buildOptionSet`、`targets`节点下的`arkOptions`子节点中通过增加`buildProfileFields`字段实现，自定义参数通过`key-value`键值对的方式配置，其中`value`取值仅支持`number`、`string`、`boolean`类型。  
当然，该配置也可以在模块下的`build-profile.json5`中配置。优先级如下：

> 模块级target > 模块级buildOptionSet > 模块级buildOption > 工程级product > 工程级buildModeSet

这里我们添加了`version_to_servier`字段来表示当前应用适配到了哪个版本。
正常情况下，我们运行代码就可以在`${moduleName} / build / ${productName} / generated / profile / ${targetName} `目录下生成`BuildProfile.ets`文件。
也可以在命令行执行`hvigorw GenerateBuildProfile`。  
也可以选中需要编译的模块，在菜单栏选择`Build > Generate Build Profile ${moduleName}`。
也可以在菜单栏选择`Build > Build Hap(s)/APP(s) > Build Hap(s)”或“Build > Build Hap(s)/APP(s) > Build APP(s)`。

使用时可以这么用

``` TypeScript
import BuildProfile from './BuildProfile';
const VERSION_TO_SERVER: string = BuildProfile.version_to_servier;
```

## 替换模块module.json5字段的值

我们使用了某三方SDK，需要在**模块**下`module.json5`文件中添加对应的id 

``` TypeScript
{
  "module": {
    "metadata": [
      {
        "name": "xxx_APPID",
        "value": "1234567"
      }
    ],
  }
}
```
为了区分测试环境和生产环境，`xxx_APPID`配置了不一样的值，我们期望是打包时通过命令行参数来修改这个值，避免认为配置出现错误。

### 实践
使用命令行`hvigorw`打包时除了`buildMode`、`debuggable`等参数外，还支持`--config properties.key=value`进行自定义参数。并且在模块下、工程下的`hvigorfile.ts`中都可以接收到该参数。

这里我们定义了布尔类型的`online`参数来表示是否为发版包，当模块下的`hvigorfile.ts`文件中根据该字段的值来区分配置的参数。
具体代码如下，在模块下的`hvigorfile.ts`文件中：
``` TypeScript
import { hapTasks, OhosHapContext, OhosPluginId } from '@ohos/hvigor-ohos-plugin';
import { getNode } from '@ohos/hvigor'

const entryNode = getNode(__filename);
// 为此节点添加一个afterNodeEvaluate hook 在hook中修改module.json5的内容并使能
entryNode.afterNodeEvaluate(node => {
  //获取命令行参数
  let online = false
  let propertyOnline = hvigor.getParameter().getProperty('online');
  if (propertyOnline != undefined) {
    online = propertyOnline
  }
  console.log("entry online-> " + propertyOnline);

  // 获取此节点使用插件的上下文对象 此时为hap插件 获取hap插件上下文对象
  const hapContext = node.getContext(OhosPluginId.OHOS_HAP_PLUGIN) as OhosHapContext;
  // 通过上下文对象获取从module.json5文件中读出来的obj对象
  const moduleJsonOpt = hapContext.getModuleJsonOpt();
  // 修改obj对象为想要的，此处举例修改module中的deviceTypes
  let metaDateList = moduleJsonOpt['module']['metadata']
  metaDateList.forEach(element => {
    if (element['name'] === 'xxx_APPID') {
      if (online) {
        console.log('线上环境，修改xxx_APPID配置为   abcdefg')
        element['value'] = 'abcdefg'
      }else{
        console.log('测试环境，修改xxx_APPID配置为   1234567')
        element['value'] = '1234567'
      }
    }
  });

    // 将obj对象设置回上下文对象以使能到构建的过程与结果中
    hapContext.setModuleJsonOpt(moduleJsonOpt);
})
export default {
    system: hapTasks,  /* Built-in plugin of Hvigor. It cannot be modified. */
    plugins:[]         /* Custom plugin to extend the functionality of Hvigor. */
}
```
在打包构建时只需要执行
``` shell
  hvigorw clean  assembleApp -p buildMode=release --config properties.online=true
```
就可以直接替换为生产环境的配置了。因为平时开发都是直接点 IDE 中的 run 进行调试，不会传入该参数，也就不会影响文件中原本配置的值。

## 打包签名
上面也提到自定义的参数也可以在工程下的`hvigorfile.ts`接收到该参数，上面`BuildProfile`中也提到在工程下的`build-profile.json5`添加了自定义字段`online`。我们同样可以根据命令行参数替换掉。同时也将配置的测试签名文件删除，只构建产物，随后再使用命令行进行签名。

代码如下，在工程根目录下的`hvigorfile.ts`文件中
``` TypeScript
import { appTasks, OhosAppContext, OhosPluginId } from '@ohos/hvigor-ohos-plugin';
import { hvigor,getNode } from '@ohos/hvigor'
// 获取根节点
const rootNode = getNode(__filename);
// 为根节点添加一个afterNodeEvaluate hook 在hook中修改根目录下的build-profile.json5的内容并使能
rootNode.afterNodeEvaluate(node => {

    //获取命令行参数
    let online = false
    let propertyOnline = hvigor.getParameter().getProperty('online');
    if (propertyOnline != undefined) {
        online = propertyOnline
    }
    console.log("online-> " + propertyOnline);

    // 获取app插件的上下文对象
    const appContext = node.getContext(OhosPluginId.OHOS_APP_PLUGIN) as OhosAppContext;
    // 通过上下文对象获取从根目录build-profile.json5文件中读出来的obj对象
    const buildProfileOpt = appContext.getBuildProfileOpt();
    //将 BuildProfile 文件中的online值改为传入的值
    buildProfileOpt['app']['products'][0]['buildOption']['arkOptions']['buildProfileFields']['online'] = online
    if (online) {
      //清除签名文件信息
        buildProfileOpt['app']['signingConfigs'] = []
    }
    
    // 将obj对象设置回上下文对象以使能到构建的过程与结果中
    appContext.setBuildProfileOpt(buildProfileOpt);
})

export default {
    system: appTasks,  /* Built-in plugin of Hvigor. It cannot be modified. */
    plugins:[]         /* Custom plugin to extend the functionality of Hvigor. */
}

```
打包的时候，由于上架需要 app 文件，所以我们需要打 release 模式的 app 文件。测试时需要打release 模式的hap 文件：
``` shell
//先安装一下依赖
ohpm install --all
//发版包
hvigorw clean  assembleApp -p buildMode=release --config properties.online=true 
//测试包
hvigorw clean  assembleHap -p buildMode=release --config properties.online=false
```
这样我们就将`BuildProfile`文件中的`online`值改为传入的值，同时也清除了签名文件配置。

这里需要注意的是，如果执行的是`assembleApp`,则产物是在项目根目录`build/outputs/${productName}/xxx-default-unsigned.app`。如果执行的是`assembleHap`，则会在`${moduleName}/build/${productName}/outputs/${productName}/entry-default-unsigned.hap`.

下面我们对产物进行签名。
``` shell
 /Applications/DevEco-Studio.app/Contents/jbr/Contents/Home/bin/java -jar /Applications/DevEco-Studio.app/Contents/sdk/default/openharmony/toolchains/lib/hap-sign-tool.jar sign-app -keyAlias "keyAlias" -signAlg "SHA256withECDSA" -mode "localSign" -appCertFile "release.cer" -profileFile "release.p7b" -inFile "build/outputs/default/xxx-default-unsigned.app" -keystoreFile "default.p12" -outFile "xxx-default-signed.app" -keyPwd "keyPwd" -keystorePwd "keystorePwd" -signCode "1"

 /Applications/DevEco-Studio.app/Contents/jbr/Contents/Home/bin/java -jar /Applications/DevEco-Studio.app/Contents/sdk/default/openharmony/toolchains/lib/hap-sign-tool.jar sign-app -keyAlias "keyAlias" -signAlg "SHA256withECDSA" -mode "localSign" -appCertFile "debug.cer" -profileFile "debug.p7b" -inFile "entry/build/default/outputs/default/entry-default-unsigned.hap" -keystoreFile "default.p12" -outFile "entry-default-signed.hap" -keyPwd "keyPwd" -keystorePwd "keystorePwd" -signCode "1"

```

我们可以把打包签名的流程写在文件(build.sh)中，每次去执行这个文件就好了
``` shell


# 初始化build_type为release
online=true
# 解析命令行参数
while [[ $# -gt 0 ]]; do
    case $1 in
        --debug)
            online=false
            echo "需要构建测试包"
            shift
            ;;
        --release)
            # 实际上这个选项是多余的，因为默认就是release
            # 但如果你希望明确指定release以覆盖其他可能设置默认值的逻辑，可以保留
            online=true
            echo "需要构建线上包"
            shift
            ;;
        *)
            # 未知选项，打印帮助信息或错误消息
            echo "Usage: $0 [--debug|--release]"
            exit 1
            ;;
    esac
done



# 安装依赖
ohpm install --all


if [ "$online" == true ]; then
    # 打线上 app 包
    echo "Executing online release build commands..."
    hvigorw clean  assembleApp -p buildMode=release --config properties.online=true 
elif [ "$online" == false ]; then
    # 打测试 hap 包
    echo "Executing not online release build commands..."
    hvigorw clean  assembleHap -p buildMode=release --config properties.online=false
else
    # 理论上不应该走到这里，除非build_type被设置为非预期的值
    echo "Unknown build type: $build_type"
    exit 1
fi



# 签名

if [ "$online" == true ]; then
    # 打线上 app 包
    echo "签名 app 文件"
    /Applications/DevEco-Studio.app/Contents/jbr/Contents/Home/bin/java -jar /Applications/DevEco-Studio.app/Contents/sdk/default/openharmony/toolchains/lib/hap-sign-tool.jar sign-app -keyAlias "keyAlias" -signAlg "SHA256withECDSA" -mode "localSign" -appCertFile "release.cer" -profileFile "release.p7b" -inFile "build/outputs/default/xxx-default-unsigned.app" -keystoreFile "default.p12" -outFile "xxx-default-signed.app" -keyPwd "keyPwd" -keystorePwd "keystorePwd" -signCode "1"

elif [ "$online" == false ]; then
    # 打测试 hap 包
    echo "签名 hap 文件"
    /Applications/DevEco-Studio.app/Contents/jbr/Contents/Home/bin/java -jar /Applications/DevEco-Studio.app/Contents/sdk/default/openharmony/toolchains/lib/hap-sign-tool.jar sign-app -keyAlias "keyAlias" -signAlg "SHA256withECDSA" -mode "localSign" -appCertFile "debug.cer" -profileFile "debug.p7b" -inFile "entry/build/default/outputs/default/entry-default-unsigned.hap" -keystoreFile "default.p12" -outFile "entry-default-signed.hap" -keyPwd "keyPwd" -keystorePwd "keystorePwd" -signCode "1"

else
    # 理论上不应该走到这里，除非build_type被设置为非预期的值
    echo "Unknown build type: $build_type"
    exit 1
fi
```
打包时，如果需要打测试包，则执行 `build.sh --debug`，如果要打发版包，则执行`build.sh --release`