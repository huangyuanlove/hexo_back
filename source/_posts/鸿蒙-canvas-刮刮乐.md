---
title: 鸿蒙-canvas-刮刮乐
tags: [HarmonyOS]
photos:
  - null
date: 2024-08-24 21:50:41
keywords: HarmonyOS,鸿蒙应用开发,鸿蒙手机应用开发,Canvas,贝塞尔曲线
---
Android 中 canvas 能画出来的东西鸿蒙的 canvas 还画不了，不大可能吧？有个朋友问鸿蒙应用中想实现刮刮乐效果，应该咋画？这个问题，你能在 Android 上用 canvas 画出来，在鸿蒙里面用 canvas 画不出来？还是 api 不熟悉吧？
<!--more-->

## 前提
这个就比较简单了。你先这样这样，然后再那样那样就行了。  
好吧， 正式点，和Android没什么区别：  
底层放一张图片或者其他什么控件都行。上面叠一层灰色canvas，手指滑动时记录一下路径，将路径上的颜色去除就行了。和手写签名唯一的区别：签名是按路径添加，刮刮乐是按路径移除。  
会了的就不用往下看了，太长不想看的直接拖到最后看源码

## 基础

和`Android`中的`PorterDuff.Mode`类似，在鸿蒙中也提供了11种不同的画布复合模式：

1. source-over: (Default) Draws a new drawing on top of an existing canvas context. 在现有的画布上方绘制新的图形。也是默认行为。
2. source-in: The new drawing is drawn only where the new drawing overlaps the target canvas.Everything else is transparent.新的图形只在与目标画布重叠的区域绘制，其他区域为透明。
3. source-out: Draws a new drawing where it does not overlap with the existing canvas content. 在不与现有画布内容重叠的区域绘制新的图形。
4. source-atop: The new drawing is drawn only where it overlaps the content of the existing canvas. 新的图形只在与现有画布内容重叠的区域绘制。
5. destination-over: Draws a new graphic behind the existing canvas content. 在现有画布内容后面绘制新的图形。
6. destination-in: Existing canvas content remains where the new drawing overlaps the existing canvas content.Everything else is transparent.仅在新的图形与现有画布内容重叠的区域保留现有画布内容，其他区域为透明。
7. destination-out: Existing content remains where the new drawing does not overlap. 仅在新的图形不与现有画布内容重叠的区域保留现有内容。
8. destination-atop: The existing canvas retains only the part that overlaps with the new drawing,which is drawn behind the canvas content. 现有画布仅保留与新的图形重叠的部分，并位于画布内容后面绘制。
9.  lighter: The color of two overlapping shapes is determined by adding the color values. 两个重叠形状的颜色通过相加颜色值来确定。
10. copy: Only new graphics are displayed. 仅显示新的图形。
11. xor: In the image, those overlaps and other places outside of the normal drawing are transparent. 在图像中，重叠部分和正常绘制范围之外的其他区域为透明。
  
看简介也可能是一脸懵，我们直接上代码看效果。先画一个红色的圆，之后设置`globalCompositeOperation`为不同的值，再画一个蓝色的正方形，直接看效果。
具体点将，就是红色圆形半径为画布的四分之一，圆心也在画布的四分之一处。蓝色正方形边长是圆形直径，左上角与圆心重合。

``` TypeScript
@Component
struct CompositeOperationView{
  @Prop compositeOperation:string
  private settings: RenderingContextSettings = new RenderingContextSettings(true)
  private canvasRendering: CanvasRenderingContext2D = new CanvasRenderingContext2D(this.settings)
  build() {
    Canvas(this.canvasRendering).width("100%").height("100%").onReady(() => {
      //获取画布宽高
      let canvasWidth = this.canvasRendering.width
      let canvasHeight = this.canvasRendering.height

      //计算圆心
      let circleX = canvasWidth/4
      let circleY = canvasHeight/4;
      let circleCenter = Math.min(circleX,circleY)

      //画红色圆形
      this.canvasRendering.fillStyle = "#FF0000"
      this.canvasRendering.arc(circleCenter,circleCenter,circleCenter,0,Math.PI * 2, false)
      this.canvasRendering.fill()

      //设置画布复合操作方式
      this.canvasRendering.globalCompositeOperation = this.compositeOperation

      //画蓝色正方形
      this.canvasRendering.fillStyle = "#0000FF"
      this.canvasRendering.fillRect(circleCenter, circleCenter, circleCenter*2, circleCenter*2)

    })
  }
}

export{CompositeOperationView}
```

这里一共有11种复合操作方式，为了方便对比，我们把这11种方式都画在同一个屏幕上。  
因此，在`CompositeOperationView`中，我们将`compositeOperation`使用`@Prop`修饰，由父级控件传进来。  

``` TypeScript
@Entry
@Component
struct CanvasCompositeOperationPage{
  compositeOperation:string[]=["source-over","source-in","source-out","source-atop","destination-over","destination-in","destination-out","destination-atop","lighter","copy","xor"]
  build() {
    Flex({ direction: FlexDirection.Row, wrap: FlexWrap.Wrap, }){
      ForEach(this.compositeOperation,(item:string)=>{
        Column(){
          Text(item)
          CompositeOperationView({compositeOperation:item}).width("100%").height("80%")
        }.width("30%").height("20%")
      })
    }
  }
}

```
这里我们用`ForEach`循环来进行绘制，这里就不在过多介绍，不清楚的可以去看鸿蒙开发文档中的指南。  
我们最终得到这样的一张图
![](image/HarmonyOS/compositeOperation.png)
这就一目了然了，我们可以选用`destination-out`来实现刮刮乐效果

## 刮刮乐效果
就和文章开始说的一样，先放一张图，然后canvas绘制蒙层，画布复合操作模式为`destination-out`，记录手指移动路径，绘制到canvas上，结束。  
详细一点就是：
1. 外层Stack层叠布局，子控件先放一个Image然后再放一个Canvas
2. Canvas上我们可以绘制上纯颜色，也可以绘制另外一张图片
3. 画布复合操作模式为`destination-out`，
4. 在Canvas的`onTouch`事件中，当`TouchType.Move`时，在当前坐标画一个小圆。
5. 结束

按照这个思路，整体代码就是这样的

``` TypeScript
@Preview
@Component
export struct ScratchOffView {
  private settings: RenderingContextSettings = new RenderingContextSettings(true)
  private canvasRendering: CanvasRenderingContext2D = new CanvasRenderingContext2D(this.settings)
  private path: Path2D = new Path2D()

  build() {
    Stack() {
      Image($r("app.media.cat"))
      Canvas(this.canvasRendering).width("100%").height("100%").onReady(() => {
        let canvasWidth = this.canvasRendering.width
        let canvasHeight = this.canvasRendering.height

        this.canvasRendering.fillStyle = "#aaf7f7f7"
        this.canvasRendering.fillRect(0, 0, canvasWidth, canvasHeight)

        this.canvasRendering.strokeStyle = "#00000000"
        this.canvasRendering.fillStyle = "#000000"
        this.canvasRendering.globalCompositeOperation = "destination-out"


      }).onTouch((event: TouchEvent) => {

        let path = new Path2D()
        let prex = 0
        let prey = 0
        switch (event.type) {
          case TouchType.Down:
            prex = event.touches[0].x
            prey = event.touches[0].y
            path.moveTo(prex, prey)
            break
          case TouchType.Move:
            this.path.moveTo(event.touches[0].x, event.touches[0].y)
            this.path.arc(event.touches[0].x, event.touches[0].y, 30, 0, Math.PI*2, false)
            this.canvasRendering.fill(this.path)
            
            break
          case TouchType.Up:
            break;
        }
      })
    }.width("100%").height("100%")

  }
}
```
最后放张效果图吧

![](image/HarmonyOS/ScratchOffView.gif)

## 总结
其实canvas并没有那么麻烦那么困难，我们只要熟悉了鸿蒙canvas的api，在Android上能实现的功能，鸿蒙能基本都能实现。比如手写签名、贝塞尔曲线、图片翻转、九宫格图片等等。  
并不像我们想象中的那么困难，总之，先动手rua代码试试呗

