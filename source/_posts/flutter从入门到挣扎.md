---
title: flutter从入门到挣扎
tags: [Android,Flutter]
photos:
  - /image/flutter/flutter.png
date: 2019-03-13 19:22:31
keywords: 跨平台,Android,flutter
---

我司准备上Flutter了，我不喜不悲。花了大概一周的时间了解了一下。写了点小玩意练手。感觉如下：

* Flutter用的前端的布局思想，就现在看来，只能算是一个UI框架加上一些简单逻辑，一旦涉及到系统的东西，比如打开系统自带浏览器、浏览系统图库等就无能为力了，只能通过`MethodChannel`和原生交互。
* 学会Flutter并不意味着就不用了解原生开发了，如果遇到了上面的情况，要么用别人写的库，要么等谷歌出封装好的包。但就现在看来，flutter也是刚刚崭露头角，京东貌似是去年下半年才开始在主业务上使用flutter。一些三方库的质量参差不齐，能在Android上运行的到iOS上就凉凉，打debug包没问题打release包就GG

下面是自己在学习、练手的时候做的一些笔记，劝退流开始了。。。。

<!--more-->

#### 环境搭建

这个看官网教程就好，基本没啥坑，如果有的话也在iOS环境上，比如pod失效之类的

链接在这里

https://flutter.dev/docs/get-started/install

英文看不懂的可以看中文的

https://flutterchina.club/get-started/install/    不过这个更新速度有点慢。

如果环境搭不好的话，也不用看下面的了，可以放弃了。

#### 常用控件

这个用到啥搜啥就行，常用也就这几个


* TabBar
* Card
* Container
* Text
* Column
* Offstage
* FadeInImage
* Row
* Icon
* RefreshIndicator
* ListView

对了，项目中还用了一个上拉加载更多，只需加添加一个滑动监听，在ListView.builder设置一下controller就好

``` dart
if (_scrollController.position.pixels == _scrollController.position.maxScrollExtent) {
        print('滑动到了最底部');
}


@override
  Widget build(BuildContext context) {
    return RefreshIndicator(
      onRefresh: () => _loadData(false),
      child: ListView.builder(
          controller: _scrollController,
          itemCount: widgets.length + 1,
          itemBuilder: (BuildContext context, int position) {
            return _getRow(position);
          }),
    );
  }
```

至于为啥`itemCount`需要数据总数+1，是为了显示出来下面的加载中动画。

在`Widget _getRow(int i)`方法中，判断是不是要渲染不是，不是渲染数据那就是渲染加载更多的这个控件

#### 网络请求及json解析

看这里 

[https://blog.huangyuanlove.com/2019/03/13/flutter-网络请求与json解析/](https://blog.huangyuanlove.com/2019/03/13/flutter-网络请求与json解析/)

#### Android和flutter混编及互相调用

看这里

[https://blog.huangyuanlove.com//2019/03/13/flutter-Android混编及互相调用/](https://blog.huangyuanlove.com//2019/03/13/flutter-Android混编及互相调用/)

#### 项目地址

https://github.com/huangyuanlove/JanDan_flutter

效果图

![煎蛋](/image/flutter/jan_dan.gif) 



----

以上

