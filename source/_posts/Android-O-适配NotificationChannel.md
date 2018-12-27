---
title: Android O---适配NotificationChannel
tags: [Android,Android爬坑之旅]
date: 2018-12-27 19:47:25
mathjax: true
keywords: [invalid channel for service notification,startForeground,NotificationChannel]
---

继之前跪在Android N的`StrictMode`上了。现在又跪在的Android O 的NotificationChannel上了

场景如下：

某些场景中需要上传图片，选择图片或者拍照时使用系统的图库会将自己的app置于后台，若选择图片的时间过长，则可能会导致自己的app会杀死。看了一下传承下来的代码，是在这种情况下发送一个前台通知`startForeground`,使此服务在前台运行，
