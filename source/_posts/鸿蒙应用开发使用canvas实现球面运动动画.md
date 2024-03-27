---
title: 鸿蒙应用开发使用canvas实现球面运动动画
tags: [HarmonyOS]
date: 2023-12-24 11:38:04
keywords: HarmonyOS,鸿蒙应用开发,鸿蒙手机应用开发
---

### 吐槽

习惯了 Android 的 Canvas,用鸿蒙的 canvas 多少有点别扭
效果图
![canvas_ball_animation.gif](image/HarmonyOS/canvas_ball_animation.gif)
上面的图是用 transform 属性做的动画  
下面的图是用 canvas 画的，参考自https://mp.weixin.qq.com/s/p_gy8s1SqPUTAa3wCIk7FQ

### 原理

众所周知，我们在手机或者平板上看到的 3D 动画只是在二维的投影，我们只需要计算好运动物体的大小和位置的对应关系，就可以实现类似 3D 的效果。想要了解具体的算法以及映射关系，可以阅读原文。
根据参考文章中的计算方式，我们只需要移植一下就行。这里是根据`总结`中的代码实现的
原文中的关键代码

``` Java
    double xr = Math.toRadians(5);  //绕x轴旋转则把这个值设置为大于 0
    double yr = 0;  //绕y轴旋转则把这个值设置为大于 0;  
    double zr = 0;  //绕z轴旋转则把这个值设置为大于 0;  
    
    //保存小球的位置、颜色及缩放
    static class Point {  
        private int color;  
        private float x;  
        private float y;  
        private float z;   
        private float scale = 1f;  
    }
    //pointList 保存的是随机生成的小球相关信息
    for (int i = 0; i < pointList.size(); i++) {  
  
            Point point = pointList.get(i);  
            float x = point.x;  
            float y = point.y;  
            float z = point.z;  
  
            //绕X轴旋转，乘以X轴的旋转矩阵  
            float rx1 = x;  
            float ry1 = (float) (y * Math.cos(xr) + z * -Math.sin(xr));  
            float rz1 = (float) (y * Math.sin(xr) + z * Math.cos(xr));  
  
            // 绕Y轴旋转,乘以Y轴的旋转矩阵  
            float rx2 = (float) (rx1 * Math.cos(yr) + rz1 * Math.sin(yr));  
            float ry2 = ry1;  
            float rz2 = (float) (rx1 * -Math.sin(yr) + rz1 * Math.cos(yr));  
  
            // 绕Z轴旋转,乘以Z轴的旋转矩阵  
            float rx3 = (float) (rx2 * Math.cos(zr) + ry2 * -Math.sin(zr));  
            float ry3 = (float) (rx2 * Math.sin(zr) + ry2 * Math.cos(zr));  
            float rz3 = rz2;  
  
  
            point.x = rx3;  
            point.y = ry3;  
            point.z = rz3;  
  
            // 透视除法，z轴向内的方向  
            float scale = (2 * radius) / ((2 * radius) + rz3);  
            point.scale = scale;  
            //到这里就完成了小球位置的计算，接下来就是需要定时更新上面xr、yr、zr的值就可以实现小球沿球面运动了
  
        }

```

### 鸿蒙 transform 实现
清楚了原理及计算方式，实现起来就简单了
先 stack 堆叠两个圆球，小球需要不断运动，x、y、z需要一直变化，使用`@State`修饰一下。值的变化过程就用上面原文中的计算方法。定时更新就用`setInterval`,组件的位移变化给我们提供了`transform`方法，需要一个`matrix4`对象，移动变化也不需要我们去填充矩阵，有对应的`translate`方法，组合起来代码如下：
``` TypeScript
import matrix4 from '@ohos.matrix4'

@Preview
@Entry
@Component
struct AnimationOfSphericalPaths {
  @State angleX: number = 0
  @State translateX: number = 0
  @State translateY: number = 0
  @State translateZ: number = 0
  @State ballRadius: number = 30
  private timerInterval: number = 0
  private radius = 200
  private a = 90
  private b = 0


  toRadians(degrees): number {
    return degrees * (Math.PI / 180)
  }

  changeAngle() {
    this.b += 3
    // this.a += 3
    if (this.a > 360) {
      this.a = this.a - 360
    }
    if (this.b > 360) {
      this.b = this.b - 360
    }

    this.translateX = this.radius * Math.cos(this.toRadians(this.a)) * Math.sin(this.toRadians(this.b)) * 2
    this.translateY = this.radius * Math.sin(this.toRadians(this.a)) * Math.sin(this.toRadians(this.b)) * 2
    this.translateZ = this.radius * Math.cos(this.toRadians(this.b)) * 2

    this.ballRadius = this.translateZ / this.radius * 10 + 50

    console.error(`translateX-> ${this.translateX} ,translateY-> ${this.translateY} ,translateZ-> ${this.translateZ}`)

  }

  build() {
    Column() {
      Stack() {
        Circle({ width: this.radius * 2, height: this.radius * 2 }).fill(Color.Yellow)
        Circle({ width: this.ballRadius, height: this.ballRadius }).fill(Color.Pink)
          .transform(matrix4.identity().translate({ x: this.translateX, y: this.translateY, z: this.translateZ }))
      }.alignContent(Alignment.Center).width(this.radius * 2).height(this.radius * 2)

      Row() {
        Button("start")
          .onClick(() => {
            if (this.timerInterval > 0) {
              return
            }

            this.timerInterval = setInterval(function(){

              this.changeAngle()

            }.bind(this), 20)
          })
        Button("stop").onClick(() => {
          clearInterval(this.timerInterval)
          this.timerInterval = 0
        }).margin({ left: 48 })
      }.margin({ top: 48 })

    }.width("100%").height("100%").backgroundColor(Color.White)
  }

}
```


### 鸿蒙canvas实现

需要注意的是，鸿蒙里面的 math 包下没有`toRadians`方法，需要我们自己实现一下
``` TypeScript
toRadians(degrees): number {
  return degrees * (Math.PI / 180)
}
```
定时更新这里用了`setInterval`  
鸿蒙的 canvas 中也没有画笔的概念，需要设置`RenderingContextSettings`实例的填充方式及填充颜色  
鸿蒙的 canvas 中也没有 drawcircle 方法，这里使用的是`Path2D`对象中画圆弧方法(arc)然后填充颜色的方式，需要注意是`Path2D.arc()`中的角度单位  

整体流程如下
1. 先生成随机的小球，分布在一个圆上generateBall()
2. 计算小球的缩放比例及位置calculateRotateValue()
3. 对小球排序，z 轴越大，越靠近我们，小球越大，越要遮盖住其他小球，越要最后画
4. 所以先画背面的，再画正面的。这里的正面和背面是相对于中间的大球来说的，为了透视效果，背面的小球加上透明度，正面的小球不透明
5. 定时更新就可以了

下面是全部代码
``` TypeScript
@Preview
@Component
export struct CanvasBallAnimation {
  private settings: RenderingContextSettings = new RenderingContextSettings(true)
  private canvasRendering: CanvasRenderingContext2D = new CanvasRenderingContext2D(this.settings)
  private canvasWidth = 0
  private canvasHeight = 0
  @State canvasBallRoundRadius: number = 0
  private canvasBallAnimationTimer: number = 0
  private startDegree: number = 5
  private xr: number = this.toRadians(this.startDegree)
  private yr: number = 0
  private zr: number = 0
  private pointList: Point[] = []

  toRadians(degrees): number {
    return degrees * (Math.PI / 180)
  }

  canvasBallAnimation() {
    this.startDegree += 5;
    if (this.startDegree > 360) {
      this.startDegree -= 360
    }

    this.canvasRendering.clearRect(0, 0, this.canvasWidth, this.canvasHeight)
    this.calculateRotateValue()
    //排序，先画背面的，再画正面的
    this.pointList.sort(this.comparator)
    this.drawFrontBall()

    //在中间画一个大圆
    this.drawCenterBall()
    this.drawBackBall()


  }

  build() {
    Column() {


      Canvas(this.canvasRendering)
        .width("100%")
        .height("40%")
        .onAreaChange((oldValue: Area, newValue: Area) => {
          this.canvasWidth = parseInt(newValue.width.toString())
          this.canvasHeight = parseInt(newValue.height.toString())

          //小球运动的半径
          this.canvasBallRoundRadius = Math.min(this.canvasWidth, this.canvasHeight) / 3
          // this.canvasRendering.translate(this.canvasWidth / 2, this.canvasHeight / 2)
          this.generateBall()
          this.calculateRotateValue()
          //排序，先画背面的，再画正面的
          this.pointList.sort(this.comparator)
          this.drawFrontBall()

          //在中间画一个大圆
         this.drawCenterBall()

          this.drawBackBall()


        })
      Row() {
        Button("canvas ball start").onClick(() => {
          if (this.canvasBallAnimationTimer > 0) {
            return
          }
          this.canvasBallAnimationTimer = setInterval(function(){
            this.canvasBallAnimation()
          }.bind(this), 20)
        })
        Button("canvas ball end").onClick(() => {
          clearInterval(this.canvasBallAnimationTimer)
          this.canvasBallAnimationTimer = 0
        })
      }
    }
  }

  comparator(left: Point, right: Point): number {
    if (left.z - right.z < 0) {
      return 1;
    }
    if (left.z == right.z) {
      return 0;
    }
    return -1;

  }

  randomColor(): string {
    let r: string = Math.floor(Math.random() * 256).toString(16)
    let g: string = Math.floor(Math.random() * 256).toString(16)
    let b: string = Math.floor(Math.random() * 256).toString(16)
    let result = `${r}${g}${b}`
    console.error(`随机颜色--> ${result}`)
    return result
  }

  generateBall() {
    if (this.pointList.length == 0) {
      let maxBallCount = 10
      this.pointList = []
      for (let i = 0; i < maxBallCount; i++) {

        let v = -1.0 + (2.0 * i - 1.0) / maxBallCount;
        if (v < -1.0) {
          v = 1.0
        }

        let delta = Math.acos(v)
        let alpha = Math.sqrt(maxBallCount * Math.PI) * delta
        let point = new Point()
        point.x = this.canvasBallRoundRadius * Math.cos(alpha) * Math.sin(delta)
        point.y = this.canvasBallRoundRadius * Math.sin(alpha) * Math.sin(delta)
        point.z = this.canvasBallRoundRadius * Math.cos(delta)
        point.color = this.randomColor()
        this.pointList.push(point)
      }
    }
  }

  calculateRotateValue() {
    for (let i = 0; i < this.pointList.length; i++) {

      let point = this.pointList[i];
      let x = point.x;
      let y = point.y;
      let z = point.z;

      //绕X轴旋转，乘以X轴的旋转矩阵
      let rx1 = x;
      let ry1 = (y * Math.cos(this.xr) + z * -Math.sin(this.xr));
      let rz1 = (y * Math.sin(this.xr) + z * Math.cos(this.xr));

      // 绕Y轴旋转,乘以Y轴的旋转矩阵
      let rx2 = (rx1 * Math.cos(this.yr) + rz1 * Math.sin(this.yr));
      let ry2 = ry1;
      let rz2 = (rx1 * -Math.sin(this.yr) + rz1 * Math.cos(this.yr));

      // 绕Z轴旋转,乘以Z轴的旋转矩阵
      let rx3 = (rx2 * Math.cos(this.zr) + ry2 * -Math.sin(this.zr));
      let ry3 = (rx2 * Math.sin(this.zr) + ry2 * Math.cos(this.zr));
      let rz3 = rz2;


      point.x = rx3;
      point.y = ry3;
      point.z = rz3;

      // 透视除法，z轴向内的方向
      let scale = (2 * this.canvasBallRoundRadius) / ((2 * this.canvasBallRoundRadius) + rz3);
      point.scale = scale;
    }
  }

  drawFrontBall() {
    for (let i = 0; i < this.pointList.length; i++) {
      let point = this.pointList[i];
      if (point.z > 0) {
        this.drawBall(point)
      } else {
        break;
      }
    }
  }

  drawBall(point: Point) {
    if (point.scale > 1) {
      this.canvasRendering.fillStyle = `#FF${point.color}`
    } else {
      let fillColor = `#${ Math.round(point.scale * 255).toString(16)}${point.color}`
      this.canvasRendering.fillStyle = fillColor
      console.error("填充颜色-->" +fillColor)
    }

    let ballPath2D = new Path2D()
    ballPath2D.arc(point.x * point.scale +this.canvasWidth / 2 , point.y * point.scale + this.canvasHeight / 2, 5 + 25 * point.scale, 0, Math.PI * 2)
    this.canvasRendering.beginPath()
    this.canvasRendering.fill(ballPath2D)
  }

  drawBackBall() {
    for (let i = this.pointList.length - 1; i >= 0; i--) {
      let point = this.pointList[i];
      if (point.z <= 0) {
        this.drawBall(point)
      } else {
        break;
      }
    }
  }
  drawCenterBall(){
    let circlePath2D = new Path2D()
    circlePath2D.arc(this.canvasWidth/2,this.canvasHeight/2,this.canvasBallRoundRadius,0,Math.PI * 2)
    this.canvasRendering.beginPath()

    let radialGradient = this.canvasRendering.createRadialGradient(this.canvasWidth/2,this.canvasHeight/2,0,this.canvasWidth/2,this.canvasHeight/2,this.canvasBallRoundRadius)
    radialGradient.addColorStop(0.0,"#ff0000")
    radialGradient.addColorStop(0.3,"#aaec5533")
    radialGradient.addColorStop(0.9,"#11000000")
    this.canvasRendering.fillStyle=radialGradient
    this.canvasRendering.fill(circlePath2D)
  }
}


class Point {
  color: string
  x: number
  y: number
  z: number
  scale: number = 1
}
```


