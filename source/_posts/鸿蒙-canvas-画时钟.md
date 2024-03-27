---
title: 鸿蒙--canvas 画时钟
tags: [HarmonyOS]
date: 2024-03-27 11:01:33
keywords: HarmonyOS,鸿蒙应用开发,鸿蒙手机应用开发,Canvas,贝塞尔曲线
---

### 前言
你在 Android 上能画出来的东西，在鸿蒙上画不出来？  
画个时钟嘛，有啥难的？  
你行你上！  
给钱就上！  
给钱？早说嘛，来来来，现在就画

### 准备
画时钟需要画哪些元素？  
圆圈、直线，没了，就这些，临时看一下canvas 相关的 api，这不都有么？直接画。  
看看需要用的方法  
``` TypeScript
arc(x: number, y: number, radius: number, startAngle: number, endAngle: number, counterclockwise?: boolean): void
```
看下参数含义
参数               | 类型      | 必填 | 默认值   | 描述         |
| :--------------- | :------ | :- | :---- | :--------- |
| x                | number  | 是  | 0     | 弧线圆心的x坐标值。 |
| y                | number  | 是  | 0     | 弧线圆心的y坐标值。 |
| radius           | number  | 是  | 0     | 弧线的圆半径。    |
| startAngle       | number  | 是  | 0     | 弧线的起始弧度。   |
| endAngle         | number  | 是  | 0     | 弧线的终止弧度。   |
| counterclockwise | boolean | 否  | false | 是否逆时针绘制圆弧。

弧度制，一圈是 2π，这个需要注意一下，还有endAngle，是终止弧度，而不是需要画多少弧度，浅浅的尝试。  
``` TypeScript

struct ClockViewTest {
  private settings: RenderingContextSettings = new RenderingContextSettings(true)
  private canvasRendering: CanvasRenderingContext2D = new CanvasRenderingContext2D(this.settings)

  build() {
    Canvas(this.canvasRendering).width("100%").height("100%")
      .onReady(() => {

        let width = this.canvasRendering.width
        let height = this.canvasRendering.height

        let centerX = width / 2
        let centerY = height / 2
        //取长宽中小的一个做直径
        let maxRadius = Math.min(width, height) / 2

        //设置线的粗细
        this.canvasRendering.lineWidth = 4

        this.canvasRendering.arc(centerX, centerY, maxRadius-50, 0, 1, false)
        //设置线的颜色
        this.canvasRendering.strokeStyle = "#ff0000"
        this.canvasRendering.stroke()

        this.canvasRendering.beginPath()
        this.canvasRendering.arc(centerX, centerY, maxRadius - 30, 0, 1, true)
        //设置线的颜色
        this.canvasRendering.strokeStyle = "#00ff00"
        this.canvasRendering.stroke()

      })
  }
}
```
效果是这样的：  

![image.png](image/HarmonyOS/canvas_circle.png)  
画直线就不用多说了，开干~~  

### 分析

#### 组成部分

指针是需要根据时间变化来转动的，表盘画好一次就不需要重绘了，偷个懒，搞两个 canvas 摞起来，底层画表盘，上层画指针，时间变了只重画上层指针就行了。  

#### 数值计算
简单的三角函数，但要注意是弧度制，数值别搞错了。  
另外需要注意的是画布左上角坐标是(0,0)，右下角坐标为(width,height)。  

#### 过程

1. 先画一大一小两个圆圈组成一个圆环。  
2. 再划线把圆环均分 60 份，每 5 条线加粗一下。  
3. 再把圆周分成 12 份，对应位置画上1~12 数字。  
4. 获取当前时间，计算出指针位置，划线。  
5. 定时更新指针位置。  
6. 结束。 

### 开始

#### 第一步 画圆环
``` TypeScript
Canvas(this.canvasRendering).width("100%").height("100%")
  .onReady(() => {

    let width = this.canvasRendering.width
    let height = this.canvasRendering.height

    let centerX = width / 2
    let centerY = height / 2
    //取长宽中小的一个做直径
    let maxRadius = Math.min(centerX, centerY)

    //留一些外边距
    let outerCircleRadius = maxRadius - 20
    this.canvasRendering.strokeStyle = "#1b91e0"
    this.canvasRendering.lineWidth = 2
    //最中间的小圈圈
    this.canvasRendering.arc(centerX, centerY, 10, 0, Math.PI * 2, false)
    this.canvasRendering.stroke()

    //画内圈
    this.canvasRendering.beginPath()
    let innerCircleRadius = outerCircleRadius -20
    this.canvasRendering.arc(centerX, centerY, innerCircleRadius, 0, Math.PI * 2, false)
    this.canvasRendering.stroke()

    //画外圈
    this.canvasRendering.beginPath()
    this.canvasRendering.arc(centerX, centerY, outerCircleRadius, 0, Math.PI * 2, false);
    this.canvasRendering.stroke()
    
  })
```
效果图  

![image.png](image/HarmonyOS/canvas_ring.png)

看着还行，颜色和粗细大家自己调。

#### 第二步 画格子
``` TypeScript
//画 60 个格子，5 的倍数则线条粗一些
let perMinuteDegree = Math.PI * 2 / 60
for (let i = 1;i <= 60; i++) {
  //结束坐标，也就是在外圆上的点
  let endX = centerX + Math.sin(i * perMinuteDegree) * outerCircleRadius
  let endY = centerY + Math.cos(i * perMinuteDegree + Math.PI) * outerCircleRadius
  //起始坐标，也就是在内圆上的点
  let startX = centerX + Math.sin(i * perMinuteDegree) * innerCircleRadius
  let startY = centerY + Math.cos(i * perMinuteDegree + Math.PI) * innerCircleRadius

  this.canvasRendering.strokeStyle = "#000000"
  let path2D = new Path2D()
  path2D.moveTo(startX, startY)
  path2D.lineTo(endX, endY)
  if (i % 5 == 0) {
    this.canvasRendering.lineWidth = 6
  } else {
    this.canvasRendering.lineWidth = 2
  }
  this.canvasRendering.stroke(path2D)
}
```
效果图  

![image.png](image/HarmonyOS/canvas_ring_clock.png)
马马虎虎，不太好看。  
这里需要注意一下，画布是以垂直向下为 Y 轴的正方向，计算时加了 `Math.PI` 弧度纠正一下  

#### 第三步 画数字

``` TypeScript
//画 1~12 数字圆形分布
this.canvasRendering.font = "40px"
let perNumberDegree = Math.PI * 2 / 12
let numberRadius = outerCircleRadius - 40

for (let i = 1;i <= 12; i++) {
  let x = centerX + Math.sin(i * perNumberDegree) * numberRadius
  let y = centerY + Math.cos(i * perNumberDegree + Math.PI) * numberRadius
  let text: string = i + ""

  this.canvasRendering.fillStyle = "#000000"
  let textMetrics: TextMetrics = this.canvasRendering.measureText(text)
  //填充文字时，传入的坐标是文字的左下角坐标
  this.canvasRendering.fillText(text, x-textMetrics.width/2 , y+textMetrics.height/2)
  //把下面这两行注释掉就没有小方块了
  this.canvasRendering.fillStyle = "#aaff6134"
  this.canvasRendering.fillRect(x,y,textMetrics.width,textMetrics.height)

}
```
效果图

![image.png](image/HarmonyOS/canvas_ring_clock_number.png)
图上的方块是为了对比画文字和画方块的坐标区别展示出来的：填充文字时传入的坐标是`文字左下角的坐标`，而画方块时是传入的`方块左上角坐标`,这里注意一下就好了，代码中测量了一下文字宽高，粗暴的做了一下纠偏。  

#### 第四、五步 画指针&定时更新
上面也说要把指针画在另外一个 `canvas` 上，减少一下绘制时的内容，没做对比，也不知道有没有作用。  
准备另外个画布，把两个画布用 `Stack` 包一下。  
``` TypeScript
Canvas(this.canvasRenderingClock).width("100%").height("100%").onReady(() => {
  this.timer = setInterval(function(){
    let date: Date = new Date()
    this.minute = date.getMinutes()
    this.hour = date.getHours()
    this.second = date.getSeconds()
    this.draw()
  }.bind(this), 500)
})
```
这里需要把第一块代码中的 `innerCircleRadius` 变量提到外部，作为类成员两个画布共用一下，主要是计算指针终点坐标用的。`centerX` 和 `centerY` 无所谓，只要两个画布对齐了，用哪个都行，这里还是提到了外部，用的第一块画布的。  

``` TypeScript
private draw() {
  清空一下画布
  this.canvasRenderingClock.clearRect(0, 0, this.centerX * 2, this.centerY * 2)

  //画秒针
  //计算秒针的角度
  let secondDegree = Math.PI * 2 / 60 * this.second
  let secondStartX = this.centerX
  let secondStartY = this.centerY
  let secondEndX = this.centerX + Math.sin(secondDegree) * this.innerCircleRadius
  let secondEndY = this.centerY + Math.cos(secondDegree + Math.PI) * this.innerCircleRadius
  let secondPath = new Path2D()
  secondPath.moveTo(secondStartX, secondStartY)
  secondPath.lineTo(secondEndX, secondEndY)
  this.canvasRenderingClock.lineWidth = 2
  this.canvasRenderingClock.stroke(secondPath)


  //画分针 颜色弄点透明度，要不然重合的时候看不清楚
  //秒针走一圈，分针走一格，其实可以忽略不计
  let minuteDegree = Math.PI * 2 / 60 * this.minute
  let minuteStartX = this.centerX
  let minuteStartY = this.centerY
  let minuteEndX = this.centerX + Math.sin(minuteDegree) * (this.innerCircleRadius / 5 * 4)
  let minuteEndY = this.centerY + Math.cos(minuteDegree + Math.PI) * (this.innerCircleRadius / 5 * 4)
  let minutePath = new Path2D()
  minutePath.moveTo(minuteStartX, minuteStartY)
  minutePath.lineTo(minuteEndX, minuteEndY)
  this.canvasRenderingClock.strokeStyle = "#aa1b91e0"
  this.canvasRenderingClock.lineWidth = 4
  this.canvasRenderingClock.stroke(minutePath)

  //画时针
  //分针走一圈，时针走 5 小格
  let hourDegree = Math.PI * 2 / 12 * this.hour + this.minute / 60 * Math.PI * 2 / 12
  let hourStartX = this.centerX
  let hourStartY = this.centerY
  let hourEndX = this.centerX + Math.sin(hourDegree) * (this.innerCircleRadius / 4 * 3)
  let hourEndY = this.centerY + Math.cos(hourDegree + Math.PI) * (this.innerCircleRadius / 4 * 3)
  let hourPath = new Path2D()
  hourPath.moveTo(hourStartX, hourStartY)
  hourPath.lineTo(hourEndX, hourEndY)
  this.canvasRenderingClock.lineWidth = 6
  this.canvasRenderingClock.strokeStyle = "#aa39d167"
  this.canvasRenderingClock.stroke(hourPath)
}
```
计算指针角度的时候也偷懒了，时针只考虑了当前分钟数，没有考虑秒数，实际差不多，先这样吧。  

#### 最后一步

效果图

![image.png](image/HarmonyOS/canvas_clock_finish.png)
就先这样吧，勉勉强强，可以自己调调颜色，调调样式，或者搞一些图片来代替这些元素也行。  

源码在这里 [github](https://github.com/huangyuanlove/HelloArkUI/blob/main/entry/src/main/ets/pages/playground/AlarmClockPage.ets) , 
[gitee](https://gitee.com/huangyuan/HelloArkUI/blob/main/entry/src/main/ets/pages/playground/AlarmClockPage.ets)

仓库地址：https://github.com/huangyuanlove/HelloArkUI  

https://gitee.com/huangyuan/HelloArkUI

---- 
以上



