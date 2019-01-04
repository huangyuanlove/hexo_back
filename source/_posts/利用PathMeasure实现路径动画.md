---
title: 利用PathMeasure实现路径动画
tags: [Android,animator]
photos:
  - "http://www.baidu.com"
date: 2019-01-04 21:47:21
keywords: Android,PathMeasure,动画,路径动画
---
我们可以利用路径动画实现很多好玩的东西，比如上面图中的类似支付宝支付完成的动画。
主要用到了`PathMeasure`,`ValueAnumator`这两个类
<!--more-->

#### PathMeasure
类似于一个计算器，可以计算一些和路径相关的东西。
两种初始化方式：
``` java
PathMeasure pathMeasure = new PathMeasure();
pathMeasure.setPath(path,false);
```
或者
``` java 
PathMeasure pathMeasure = new PathMeasure(path,false);
```

##### getLength()
用来获取路径长度，并且获取到的是当前曲线的长度，而不是整个Path的长度
``` java
@Override
protected void onDraw(Canvas canvas) {
    super.onDraw(canvas);
    canvas.translate(150, 150);
    path.addRect(-50,-50,50,50,Path.Direction.CW);
    path.addRect(-100,-100,100,100,Path.Direction.CW);
    path.addRect(-120,-120,120,120,Path.Direction.CW);

    canvas.drawPath(path, paint);
    pathMeasure.setPath(path,false);
    do{
    	Log.e("huangyuan",pathMeasure.getLength()+"");
    }while (pathMeasure.nextContour());

}
```
> 2019-01-04 21:58:28.408 5231-5231/huangyuanlove.com.customwidget E/huangyuan: 400.0
2019-01-04 21:58:28.408 5231-5231/huangyuanlove.com.customwidget E/huangyuan: 800.0
2019-01-04 21:58:28.408 5231-5231/huangyuanlove.com.customwidget E/huangyuan: 960.0

pathMeasure.nextContour()得到的曲线的顺序与添加到Path中的顺序相同

##### getSegment()
函数定义
> public boolean getSegment(float startD, float stopD, Path dst, boolean startWithMoveTo)

懒得翻译，自己看吧
``` java
/**
    * Given a start and stop distance, return in dst the intervening
    * segment(s). If the segment is zero-length, return false, else return
    * true. startD and stopD are pinned to legal values (0..getLength()).
    * If startD >= stopD then return false (and leave dst untouched).
    * Begin the segment with a moveTo if startWithMoveTo is true.
*/
```
我们可以这么用
``` java
    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        canvas.translate(150, 150);
        path.addRect(-50,-50,50,50,Path.Direction.CW);
        Path dst = new Path();
        pathMeasure.setPath(path,false);
        pathMeasure.getSegment(0,150,dst,true);
        canvas.drawPath(dst,paint);
    }

```
得到：
![getSegment](/image/Android/PathMeasure/dst.png  "getSegment")
如果dst路径不为空
``` java
Path dst = new Path();
dst.lineTo(10,100);
```
得到
![dst路径不空](/image/Android/PathMeasure/dst_not_null.png  "dst路径不空")
如果dst路径不空，`startWithMoveTo`为false，得到
![starWithMoveToFalse，](/image/Android/PathMeasure/starWithMoveToFalse.png  "starWithMoveToFalse")

