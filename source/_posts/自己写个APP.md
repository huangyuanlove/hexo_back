---
title: 自己写个APP
date: 2016-12-01 16:00:22
tags: [Android,自己写个APP,J2EE]
---
自己动手写个App吧，包括前端(Android,Web)和后台(J2EE)
<!--more-->
继上次自己写了个应用后，现在又想再写一个好点的，因为上次写的太矬了：
1. 后端用jsp+servlet+MongoDB，简单粗暴。
2. 前端使用Volley网络请求，以及从github上找的布局菜单框架，智能回复接入的图灵机器人
3. 还有一堆bug
4. 勉强在应用宝上线了，但是没啥意思。
现在又想再写一个APP了，写的规范点吧，主要想法如下：
#### Android向
1. 主体框架使用 'dataBinding' 和 'mvvm' 模式 
2. 网络请求使用'okHttp'框架
3. 列表使用主要使用'RecyclerView'
4. 客户端数据库框架使用'greendao'
5. 加密使用'md5'
6. 图片下载缓存使用'universal image loader' android 三大图片缓存原理、特性对比
7. json解析使用'Gson'
8. 广播使用'BusEvent'
9. 使用'腾讯BugLy'统计崩溃信息
10. 打算加入'热修复'功能360热修复demo,原理自行搜索吧,这个功能只是但算加入的.
11. 界面风格使用 Material Design 
#### 后端向--J2EE
1. 主题框架使用 ssm
2. 数据库使用MySql

#### 其他
主要功能和QQ空间差不多吧，记录自己的心情以及聊天之类的。
打算把自己的开发过程记录下来，主要是总结之类的吧，不定期更新。