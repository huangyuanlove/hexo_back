---
title: 鸿蒙-多包开发:HAP、HAR和HSP
tags: [HarmonyOS]
date: 2025-02-27 10:01:17
keywords: HarmonyOS,鸿蒙应用开发,鸿蒙手机应用开发,多包开发,HAP,HAR,HSP,静态共享库,动态共享库,Static Library,Shared Library
---

支持模块化开发：将每个功能模块作为一个独立的 Module进行开发，Module 中可以包含源码、资源文件、第三方库、配置文件等，每一个 Module可以独立编译，实现特定的功能
支持多设备适配：每个 Module可以单独配置所支持的设备类型，那么在应用市场分发应用包时，也能够根据设备类型做精准的筛选和匹配，从而将不同的包合理的组合和部署到对应的设备上。

## Module 类型
### Ability类型的 Module
用于实现应用的功能和特性，每一个 Ability 类型的 Module编译后，会生成一个以`.hap`为后缀的文件，被称为HAP(Harmony Abilit Package)。可以被独立安装和运行，是应用安装的基本单位，一个应用中可以包含一个或多个HAP 包。其中又可以分为两种类型：
entry 类型的Module：应用主模块，包含应用的入口界面、入口图标和主功能特性，编译后生成entry 类型的 HAP。每一个应用分发到同一类型的设备上的应用程序包，只能包含唯一一个entry 类型的HAP，也可以不包含。
feature类型的 Module：应用的动态特性模块，编译后生成feature 类型的 HAP，一个应用可以包含一个或多个feature 类型的HAP，也可以不包含。

### Library类型的 Module

用于实现代码和资源的共享，同一个Library 类型的 Module可以被其他的 Module多次引用。Library 类型的 Module分为 Static和 Shared 两种类型，编译后会生成共享包。
Static Library：静态共享库。编译后会生成一个以`.har`为后缀的文件，也就是静态共享包HAR(Harmony Archive)。
Shared Library：动态共享库，编译后会生成一个已`.hsp`为后缀的文件，也就是动态共享包HSR(Harmony Shared Package)


### 区别

实际上，Shared Library编译后除了会生成一个 hsp 文件外，还会生成一个 har文件，这个 har 文件中包含了 hsp对外导出的接口，应用中其他模块需要通过har 文件来引用 hsp功能，为了表述方便，通常认为Shared Library编译后生成 HSP。来看一下二者的区别。

对于 HAR 文件来讲，其中的代码和资源跟随使用方编译，如果有多个使用方，他们的编译产物中会存在多份拷贝。建议开启混淆能力，保护代码资产。除了支持应用内引用，还可以独立打包发布，供其他应用引用。
对于 HSP 文件来讲，其中的代码和资源可以独立编译，运行时在一个进程中代码也只会存在一份。该文件一般随应用进行打包，当前支持应用内和集成态HSP。应用内HSP只支持应用内引用，集成态HSP支持发布到ohpm私仓和跨应用引用。

### HAP包限制
* 不支持导出接口和ArkUI组件，给其他模块使用。

* 多HAP场景下，App Pack包中同一设备类型的所有HAP中必须有且只有一个Entry类型的HAP，Feature类型的HAP可以有一个或者多个，也可以没有。

* 多HAP场景下，同一应用中的所有HAP的配置文件中的bundleName、versionCode、versionName、minCompatibleVersionCode、debug、minAPIVersion、targetAPIVersion、apiReleaseType相同，同一设备类型的所有HAP对应的moduleName标签必须唯一。HAP打包生成App Pack包时，会对上述参数配置进行校验。

* 多HAP场景下，同一应用的所有HAP、HSP的签名证书要保持一致。上架应用市场是以App Pack形式上架，应用市场分发时会将所有HAP从App Pack中拆分出来，同时对其中的所有HAP进行重签名，这样保证了所有HAP签名证书的一致性。在调试阶段，开发者通过命令行或DevEco Studio将HAP安装到设备上时，要保证所有HAP签名证书一致，否则会出现安装失败的问题。

### HAR包限制
* HAR不支持在设备上单独安装/运行，只能作为应用模块的依赖项被引用。
* HAR不支持在配置文件中声明ExtensionAbility组件，但支持UIAbility组件。
>说明
>如果使用startAbility接口拉起HAR中的UIAbility，接口参数中的moduleName取值需要为依赖该HAR的HAP/HSP的moduleName。

* HAR不支持在配置文件中声明pages页面，但是可以包含pages页面，并通过Navigation跳转的方式进行跳转。
* HAR不支持引用AppScope目录中的资源。在编译构建时，AppScope中的内容不会打包到HAR中，因此会导致HAR资源引用失败。
* HAR可以依赖其他HAR，但不支持循环依赖，也不支持依赖传递。

### HSP包限制
* HSP不支持在设备上单独安装/运行，需要与依赖该HSP的HAP一起安装/运行。HSP的版本号必须与HAP版本号一致。
* HSP不支持在配置文件中声明ExtensionAbility组件，但支持在配置文件中声明UIAbility（除入口ability外）组件。
* HSP可以依赖其他HAR或HSP，但不支持循环依赖，也不支持依赖传递。
