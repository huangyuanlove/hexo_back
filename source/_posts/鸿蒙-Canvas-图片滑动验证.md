---
title: 鸿蒙--Canvas 图片滑动验证
tags: [HarmonyOS]
date: 2024-11-28 22:55:23
keywords: HarmonyOS,鸿蒙应用开发,鸿蒙手机应用开发,Canvas,图片滑动验证码

---



群里有朋友问图片滑块验证码怎么做，就是一张图上扣出来一块，然后拖动这一小块完成拼图。
第一个想法就是偷懒一下：直接让设计在图片上抠出来一小块，把这两个图片和抠图的坐标一块下发，用Image或者canvas自己绘制一下，监听一下手指移动，当手指抬起的时候，如果移动的坐标和抠图的坐标误差在指定范围内，就算成功。
后来说Android那边是自己处理的，下发整张图片，然后客户端自己抠图，自己处理。\
Android能做的，鸿蒙应该也能做，这时候就应该掏出来Canvas怼一波了
<!--more-->
## 过程

两个Canvas，一个使用`drawImage`画整张图片，画出来后，随机两个坐标值使用`getImageData`获取指定位置的图片内容。然后在这个区域绘制上边框或者填充颜色，告诉用户获取的是这个区域的内容。想上难度的话，不提示这个截取位置也行。\
在另外一个Canvas上使用`putImageData`将图片绘制出来，绑定一下移动手势监听，然后不断更新绘制图片的坐标。当抬起手指的时候，对比一下移动的坐标和抠图的坐标，在允许的范围内，判定为成功。\
结束。打完收工。完结撒花。

![image.png](image/HarmonyOS/slide_code.gif)  

## 绘制形状方式详细解释

先看下面不需要处理抠图的，这个简单点，我们循序渐进。

### 定义变量

两个Canvas，需要两个`CanvasRenderingContext2D`分别绘制两个Canvas上的内容。\
一个能接受的误差值。\
随机出来的抠图的横纵坐标。\
抠图的大小。

```TypeScript
private settings: RenderingContextSettings = new RenderingContextSettings(true)
private canvasRendering: CanvasRenderingContext2D = new CanvasRenderingContext2D(this.settings)
private canvasRendering2: CanvasRenderingContext2D = new CanvasRenderingContext2D(this.settings)
//允许的误差
private diffInterval: number = 10 
//随机抠图的横坐标
private clip_start_x: number = 100
//随机抠图的纵坐标
private clip_start_y: number = 100
//抠图的宽度
private clip_image_width = 120
//抠图的高度
private clip_image_height = 120
```

### 布局

这个没啥好说的，Stack里面摞两个Canvas，底部的Canvas画整个图，上面的Canvas画形状。

### 整图Canvas

这里使用的本地图片，理论上讲，使用网络图片应该也能处理。\
随机坐标时，注意减去抠图的宽度，否则万一随机出来的坐标在绘制完形状之后超出的图片范围就好玩了。\
这里随机之后绘制了一个三角形。

```TypeScript
     Canvas(this.canvasRendering).width("100%").height("100%").onReady(() => {

        //这里用的本地图片
        let imageBitMap: ImageBitmap = new ImageBitmap("pages/playground/cat.webp")
        this.canvasRendering.drawImage(imageBitMap, 0, 0)
        hilog.error(0x01, 'SlideVerificationView2', 'imageBitMap width --> ' + imageBitMap.width)
        hilog.error(0x01, 'SlideVerificationView2', 'imageBitMap height --> ' + imageBitMap.height)

        //随机两个坐标，注意不要超出图片范围
        this.clip_start_x = Math.floor(Math.random() * (imageBitMap.width - this.clip_image_width))
        this.clip_start_y = Math.floor(Math.random() * (imageBitMap.height - this.clip_image_height))

        hilog.error(0x01, 'SlideVerificationView2', 'clip_start_x --> ' + this.clip_start_x)
        hilog.error(0x01, 'SlideVerificationView2', 'clip_start_y --> ' + this.clip_start_y)

        this.canvasRendering.lineWidth = 2
        this.canvasRendering.strokeStyle = '#FFFFFF'
        //在对应的区域绘制标识，这里画了个三角形，想画其他的自己调整就好
        this.canvasRendering.moveTo(this.clip_start_x + this.clip_image_width / 2, this.clip_start_y)
        this.canvasRendering.lineTo(this.clip_start_x + this.clip_image_width, this.clip_start_y + this.clip_image_height)
        this.canvasRendering.lineTo(this.clip_start_x, this.clip_start_y + this.clip_image_height)
        this.canvasRendering.lineTo(this.clip_start_x + this.clip_image_width / 2, this.clip_start_y)
        this.canvasRendering.stroke()
      })
```

### 需要滑动的形状

我们拿到了随机的坐标后，在新的Canvas上绘制相同的形状。\
这里需要监听手指的滑动，我们使用了`priorityGesture`来绑定`PanGesture`。注意这里**滑动最小距离为5vp时识别成功**。\
这里我们限制了只能横向滑动。想加点难度的话，可以在横纵方向上都能滑动。\
最后在`onActionEnd`的时候判断一下移动的坐标是否满足条件

```TypeScript
      Canvas(this.canvasRendering2).width("100%").height("100%").onReady(() => {
      })
        //绑定优先识别手势
        .priorityGesture(
          //平移手势，滑动最小距离为5vp时识别成功。
          PanGesture()
            .onActionStart((event: GestureEvent) => {
            })
            .onActionUpdate((event: GestureEvent) => {
              //重置一下画布
              this.canvasRendering2.reset()

              //绘制形状，和整图canvas中的形状、大小一致
              this.canvasRendering2.moveTo(event.offsetX + this.clip_image_width/2, this.clip_start_y)
              this.canvasRendering2.lineTo(event.offsetX + this.clip_image_width, this.clip_start_y + this.clip_image_height)
              this.canvasRendering2.lineTo(event.offsetX, this.clip_start_y + this.clip_image_height)
              this.canvasRendering2.lineTo(event.offsetX + this.clip_image_width/2, this.clip_start_y)

              this.canvasRendering2.strokeStyle = Color.Pink
              this.canvasRendering2.lineWidth = 2
              this.canvasRendering2.stroke()

            })
            .onActionEnd((event: GestureEvent) => {
              hilog.error(0x01, 'SlideVerificationView', `onActionEnd ${event.offsetX.toString()}`)
              //判定是否成功
              if (Math.abs(event.offsetX - this.clip_start_x) < this.diffInterval) {
                promptAction.showToast({ message: '验证成功' })
              } else {
                promptAction.showToast({ message: '验证失败' })
                this.canvasRendering2.reset()
              }
            })
        )
```

这种是最简单的，不需要处理图片，只需要绘制形状就好了

## 需要处理图片的方式

比起上面这种，我们只需要多定义一个ImageData就好了

```TypeScript
private settings: RenderingContextSettings = new RenderingContextSettings(true)
private canvasRendering: CanvasRenderingContext2D = new CanvasRenderingContext2D(this.settings)
private canvasRendering2: CanvasRenderingContext2D = new CanvasRenderingContext2D(this.settings)
private diffInterval: number = 10
private clip_start_x: number = 100
private clip_start_y: number = 100
private clip_image_width = 120
private clip_image_height = 120
private imageData?: ImageData
```

### 处理抠图

在绘制整图的Canvas上调用`getImageData`获取一下抠出来的图片内容就好了。\
由于ImageData是个正方形，我们这里需要处理成三角形，我没有找到很好的方法，只能对ImageData.data属性进行处理，它是一维数组，保存了相应的颜色数据，数据值范围为0到255。

```TypeScript
      Canvas(this.canvasRendering).width("100%").height("100%").onReady(() => {

        let imageBitMap: ImageBitmap = new ImageBitmap("pages/playground/cat.webp")
        this.canvasRendering.drawImage(imageBitMap, 0, 0)

        hilog.error(0x01, 'SlideVerificationView2', 'imageBitMap width --> ' + imageBitMap.width)
        hilog.error(0x01, 'SlideVerificationView2', 'imageBitMap height --> ' + imageBitMap.height)


        this.clip_start_x = Math.floor(Math.random() * (imageBitMap.width - this.clip_image_width))
        this.clip_start_y = Math.floor(Math.random() * (imageBitMap.height - this.clip_image_height))

        hilog.error(0x01, 'SlideVerificationView2', 'clip_start_x --> ' + this.clip_start_x)
        hilog.error(0x01, 'SlideVerificationView2', 'clip_start_y --> ' + this.clip_start_y)

        this.imageData = this.canvasRendering.getImageData(this.clip_start_x, this.clip_start_y, this.clip_image_width, this.clip_image_height)

        //在对应的区域绘制标识
        this.canvasRendering.lineWidth = 2
        this.canvasRendering.fillStyle = '#66FFFFFF'
        this.canvasRendering.moveTo(this.clip_start_x + this.clip_image_width / 2, this.clip_start_y)
        this.canvasRendering.lineTo(this.clip_start_x + this.clip_image_width, this.clip_start_y + this.clip_image_height)
        this.canvasRendering.lineTo(this.clip_start_x, this.clip_start_y + this.clip_image_height)
        this.canvasRendering.lineTo(this.clip_start_x + this.clip_image_width / 2, this.clip_start_y)
        this.canvasRendering.fill()

        //将ImageData处理成三角形
        if(this.imageData){
          let width = this.imageData.width * 4
          let height = this.imageData.height
          let rate = width / height
          let widthCenter = Math.floor(width / 2)
          for (let i = 0; i < height; i++) {
            //第几行

            for (let j = 0; j < width; j++) {
              //第几列
              if (j < widthCenter - rate * i / 2) {
                this.imageData.data[i * width +j] = 0
              } else if (j > widthCenter + rate * i / 2) {
                this.imageData.data[i * width +j] = 0
              }

            }
          }
        }
      })
```

### 绘制抠出来的图

这个就更简单了，相同的绑定手势方法，相同的判定方法。\
唯一的变化就是在`onActionUpdate`回调中使用`putImageData`绘制图片

```TypeScript
    Canvas(this.canvasRendering2).width("100%").height("100%").onReady(() => {
      })
        .priorityGesture(
          PanGesture()
            .onActionStart((event: GestureEvent) => {
            })
            .onActionUpdate((event: GestureEvent) => {
              hilog.error(0x01, 'SlideVerificationView', event.offsetX.toString())
              
              if (this.imageData) {
                this.canvasRendering2.reset()
                this.canvasRendering2.putImageData(this.imageData, event.offsetX, this.clip_start_y)
              }
            })
            .onActionEnd((event: GestureEvent) => {
              hilog.error(0x01, 'SlideVerificationView', `onActionEnd ${event.offsetX.toString()}`)

              if (Math.abs(event.offsetX - this.clip_start_x) < this.diffInterval) {
                promptAction.showToast({ message: '验证成功' })
              } else {
                promptAction.showToast({ message: '验证失败' })
                this.canvasRendering2.reset()
              }
            })
        )
```

到此，我们就完成了简单的滑动图片验证的功能

## 总结

整体的流程上面也说过了，这里就不再赘述。\
我们还可以加大点难度，比如在抠图后不在原图上提示范围，让使用者自己找。\
比如我们还可以将抠出来的图镜像一下，让使用者自己找。\
比如我们还可以将抠出来的图隔像素点抽样一下。\
比如我们还可以将抠出来的图中的像素调整一下颜色。\
。。。
