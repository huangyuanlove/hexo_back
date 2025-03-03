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