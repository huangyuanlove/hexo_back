---
title: 《我的安卓爬坑之旅》--融云IM遇到的坑
date: 2016-12-01 15:20:47
tags: [Android爬坑之旅,Android,融云IM]
---
这篇博客是关于融云IM使用中遇到的问题，不算是坑，只能说是注意事项吧
<!--more-->
#### 后端向
  在自己的应用"OurStories"中打算接入IM即时通讯功能，就使用了融云提供的sdk，需要自己写后台获取Token，这个比较简单，官方给出了Demo，按照自己的习惯，把demo里面的方法封装一下就可以使用了。
#### Token向
1. 获取Token时可以在融云后台（登录自己帐号，控制台）可以设置Token有效期。
2. 在初始情况下开发环境下最多只能有100个测试用户，当达到上限后可以点击添加用户的按钮，每次添加20人上限，生产环境没有上限。
3. 开发环境和生产环境在融云后台是两套独立的环境，拥有不同的`App Key`和`App Secret`，当产品上线时不要忘记切换自己服务器和app的配置，另外，app的`App Key`和服务器端的`App Key`要一致。
#### 前端 Android向
1. 由于集成融云的聊天界面个会话列表界面都是Fragment形式，在集成的过程中，包含该Fragment的Activity要继承自**FragmentActivity**，否则在开启聊天界面的时候会报如下异常：
```java
Caused by: android.view.InflateException: Binary XML file line #6: Binary XML file line #6: Error inflating class fragment 
Caused by: android.view.InflateException: Binary XML file line #6: Error inflating class fragment 
Caused by: android.app.Fragment$InstantiationException: Trying to instantiate a class io.rong.imkit.fragment.ConversationFragment that is not a Fragment 
Caused by: java.lang.ClassCastException 
```
2. 融云在初始化的时候建议放在Applicatuon中进行，但是融云会开启3个进程，每个进程都会执行Application的OnCreate方法，建议在初始化自己的配置时检测以下进程，在自己的主进程中初始化自己的配置.
3. 千万不要忘记配置包含融云Fragment的Activity的`<intent-filter>`
4. 融云不同步也不会保存应用下的好友关系，需要自己的服务器保存
5. 注意阅读融云的开发文档，注意每一个细节
6. 有问题先去搜知识库，然后提工单。提工单的时候尽可能详细的描述自己的开发环境，遇到的问题以及异常日志。
7. 非必要情况下，不要自己去反编译出融云的sdk，然后自己使用用其中的代码。

