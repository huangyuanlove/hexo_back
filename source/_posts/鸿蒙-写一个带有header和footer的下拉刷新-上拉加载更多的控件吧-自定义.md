---
title: 鸿蒙-写一个带有header和footer的下拉刷新,上拉加载更多的控件吧-自定义
tags: [HarmonyOS]
date: 2026-01-21 08:56:08
keywords: HarmonyOS,下拉刷新,上拉加载更多,pullToRefresh,priorityGesture,PanGesture
---

## 前言

上一篇介绍了如何使用自带的 Refresh 来实现下拉刷新和添加一个额外的 ListItem 来实现上拉加载功能。这一篇我们来看下不使用这这两种方式，还能如何实现。

## 大致思路

1. 在Stack组件中放置三个子组件，自定义的下拉刷新组件、List组件、加载更多组件。
2. 下拉刷新和上拉加载更多组件通过 position 和translate属性控制位置。并且将父组件 Stack 的clip属性设置为 true，以裁剪掉超出范围的部分。
3. 并在 Stack 中通过priorityGesture绑定优先识别手势PanGesture，在上下滑动的过程中判断子组件 List 的位置，以判断
4. 根据滑动的方向和距离，做出不同的效果来提示用户
5. 注意处理各种状态，比如刷新、加载更多过程中不能再次刷新、加载更多等

先看效果：
![下拉刷新](image/HarmonyOS/pull_to_refresh.gif)
![上拉加载更多](image/HarmonyOS/pull_up_load_more.gif)

## 实现

### 整体布局

先看下页面整体的布局
![pull_refresh_and_load_more_page.png](image/HarmonyOS/pull_refresh_and_load_more_page.png)  
整个页面，除去顶部的`ActionBar`,只看剩下的刷新相关的布局，类似下面这种

```typescript
Stack() {
    Stack({ alignContent: Alignment.Top }) {
        this.header()
    }.translate({
        y: this.headerTransitionY
    }).height('100%')


    Stack({ alignContent: Alignment.Bottom }) {
        this.footer()
    }.translate({
        y: this.footerTransitionY
    }).height('100%')

    List({ space: 20, scroller: this.scroller })
    .edgeEffect(EdgeEffect.None)
    .translate({
        y: this.headerTransitionY || this.footerTransitionY
    })

}.layoutWeight(1).clip(true)
```

顶部的Stack高度设置为 100%，通过`alignContent: Alignment.Top`属性将子组件`this.header()`放在最上面。对于`this.footer()`也是同样的处理。注意这里设置的`translate`和`List`组件的`translate`。因为 List 要和 header、footer 同时滑动。

### header

但这样还不够，初始状态下我们需要将 header 向上平移它本身的高度。footer 也需要向下平移它本身的高度，将 header 和 footer 隐藏起来。

```typescript
  @LocalBuilder
  header() {
    Column() {
      Row() {
        Image($r('app.media.tab_guide_unselect')).width(20).height(20).rotate({
          centerX: "50%",
          centerY: "50%",
          angle: this.headerImageRotate
        })
        Blank().width(10).height(0)
        Text(`${this.headerText} ${this.getLastRefreshTime()}`)
      }.width("100%").alignItems(VerticalAlign.Center).justifyContent(FlexAlign.Center)

    }
    .width("100%")
    .alignItems(HorizontalAlign.Center)
    .justifyContent(FlexAlign.Center)
    .height(60)
    .backgroundColor(Color.Red)
    .onAreaChange((_, newValue) => {
      this.headerMaxTransition = newValue.height as number
    })
    .position({ top: -this.headerMaxTransition })
  }
```

对于 header，放了一个图片随着滑动进行旋转。
并且定义了一个`headerMaxTransition`来记录 header 的高度，方便我们后面操作。
随后通过position将 header 移动到绘制区域之外。

### footer

```typescript
  @LocalBuilder
  footer() {
    Column() {
      Text(`${this.footerText} ${this.getLastLoadMoreTime()}`).height(60).backgroundColor(Color.Green)
    }.width("100%").onAreaChange((_, newValue) => {
      this.footerMaxTransition = newValue.height as number
    }).position({ bottom: -this.footerMaxTransition }).backgroundColor(Color.Orange)
  }
```

对于 footer，也是同样的操作，`footerMaxTransition`记录footer 的高度，同样通过position将 footer 移动到绘制区域之外

### 滑动

我们对最外层的 Stack 的添加有限识别手势来识别垂直方向的滑动

```typescript
.priorityGesture(
    PanGesture({ direction: PanDirection.Vertical })
        .onActionStart((event: GestureEvent) => {}
        .onActionUpdate((event: GestureEvent) => {}
        .onActionEnd((event: GestureEvent) => {}
        .onActionCancel(() => {}
)
```

这样我就可以优先于子控件 List 来响应上下滑动事件了。
我们在onActionStart回调中记录按下的位置；在onActionUpdate处理并分发滑动事件，也就是判断应该滑动哪个组件；在onActionEnd中判断是否需要刷新等操作

#### onActionStart

我们在事件开始的时候记录一下按下的位置

```typescript
.onActionStart((event: GestureEvent) => {
    this.lastScroll = event.offsetY
})
```

#### onActionUpdate

先计算一下滑动的距离，然后将滑动的速度分发到`List`组件

```typescript
.onActionUpdate((event: GestureEvent) => {
    let diff = event.offsetY - this.lastScroll
    this.lastScroll = event.offsetY
    if (diff) {
    if (event.velocity) {
        if (diff > 0) {
            this.scroller.fling(event.velocity)
        } else {
            this.scroller.fling(-event.velocity)
        }

    }
    this.handleScroll(diff)
    }
})
```

这里调用的`scroller.fling`方法,其中参数`velocity`值设置为**0**，视为**异常值**，本次滚动不生效。如果值为**正数**，则向**顶部**滚动；如果值为**负数**，则向**底部**滚动。
下面看下处理滑动的`handleScroll`方法

#### handleScroll

先分类总结一下各种情况

1. 向上滑动
   - 列表在最底部
     - footer 的偏移距离已经超过了它本身的高度
       - 这时候`footer`已经完全漏出来了。这时候我们可以将阻尼系数写小一些，也就是手指滑动 10单位，控件移动2 或者3 单位；
       - 需要标记松手后需要执行加载更多操作，也就是在onActionEnd方法中处理
       - 需要将提示文案修改为`松开后加载更多`
     - footer的偏移距离没有超过它本身高度
       - 这时候`footer`已经漏出来一部分，这时候我们可以将阻尼系数稍微调大一些， 也就是手指滑动 10单位，控件移动5 或者6 单位；
       - 这时候如果松手，则需要将组件偏移距离取消
       - 取消松手时加载更多标记
       - 需要将提示文案修改为`上拉加载更多`

   - 列表不在最底部
     - header 有偏移
       - 将 header 向上平移
       - 将 header 中的图片反向旋转
   - header 没有偏移
     - 将滑动分发到 List 组件

2. 向下滑动

- 列表在最顶部
  - header 偏移距离超过它本身高度
    - header 偏移距离不变
    - 图片继续随手指滑动而旋转
    - 标记松开手时处理刷新
    - 提示文案修改为`松开后刷新`
  - header 偏移距离没有超过它本身高度
    - 文案修改为`下拉刷新`
    - 取消松开手时的刷新标记
    - 松手时将组件偏移距离取消
- 列表不在最顶部
  - footer 有偏移距离
    - 将 footer 向下偏移
  - footer 没有偏移距离
    - 将滑动分发到 List

大致就这些情况，我们来看下具体实现

```typescript
  handleScroll(offset: number) {
    if (offset < 0) {
      //向上滑动
      if (this.scroller.isAtEnd()) {
        console.error(`列表在底部，继续上拉`)
        //列表滑动到底部，继续上拉，
        if (Math.abs(this.footerTransitionY) > this.footerMaxTransition) {
          this.footerTransitionY += (offset * 0.2)
          if(this.currentState != State.Loading){
            this.footerText = '松开后加载更多'
            this.needLoadMoreOnDidScroll = true
          }

        } else {

          this.footerTransitionY += (offset * 0.3)
          if(this.currentState != State.Loading){
            this.footerText = '上拉加载更多'
            this.needLoadMoreOnDidScroll = false
          }

        }
      } else {
        console.error(`列表不在底部，继续上拉`)
        if (this.headerTransitionY > 0) {
          this.headerTransitionY += (offset * 0.3)
          this.headerImageRotate -= (offset * 1.5)
        } else {
          this.headerTransitionY = 0
          this.scroller.scrollBy(0, -offset)
        }

      }
    } else {
      //向下滑动
      if (this.scroller.currentOffset().yOffset == 0) {
        //列表滑动到顶部，继续下拉需要去处理刷新
        this.headerTransitionY += (offset * 0.3)
        this.headerImageRotate += (offset * 1.5)
        if (Math.abs(this.headerTransitionY) >= this.headerMaxTransition) {
          this.headerTransitionY = this.headerMaxTransition
          if(this.currentState != State.Loading){
            this.headerText = '松开后刷新'
            this.needRefreshOnDidScroll = true
          }

        } else {
          if(this.currentState != State.Loading){
            this.headerText = '下拉刷新'
            this.needRefreshOnDidScroll = false
          }

        }
      } else {
        if (this.footerTransitionY > 0) {
          this.footerTransitionY -= (offset * 0.3)
        } else {
          this.footerTransitionY = 0
          this.scroller.scrollBy(0, -offset)
        }


      }
    }
  }
```

#### onActionEnd

在滑动事件结束的时候，需要处理的事情有如下几个：
如果不需要刷新，则取消掉 header 的偏移。
如果需要刷新，则在刷新完成后取消掉 header 的偏移。
如果不需要加载更多，则取消掉 footer 的偏移。
如果需要加载更多，则将 footer 的偏移设置为它本的高度，加载完成后取消掉偏移
在取消偏移量的时候，为了能有更好的体验，可以使用动画做平滑过渡

```typescript
.onActionEnd((event: GestureEvent) => {
    if (event.velocityY) {
        //先处理快速滑动
        this.scroller.fling(event.velocityY)
    }
    //需要刷新 并且 当前不在加载中章台
    if (this.needRefreshOnDidScroll && this.currentState != State.Loading) {
        this.currentState = State.Loading
        this.headerText = '刷新中'
        this.needRefreshOnDidScroll = false
        //模拟网络延时
        setTimeout(() => {
            this.headerText = '刷新成功'
            this.currentState = State.Idle
            this.lastRefreshTime = systemDateTime.getTime()
            setTimeout(() => {
                this.headerTransitionY = -this.headerMaxTransition
                //500毫秒的动画，平滑的取消偏移量
                this.getUIContext().animateTo({
                  duration: 500,
                  curve: Curve.EaseOut,
                  iterations: 1,
                  playMode: PlayMode.Normal,
                  onFinish: () => {
                    console.error('刷新后恢复 play end');
                  }
                }, () => {
                  this.headerTransitionY = 0
                })

              }, 1000)
            }, 2000)
          } else {
            // 这里对应着header 没有完全漏出，松手后平滑的取消偏移量
            this.resetHeaderTransition()
          }

          //需要加载更多 且 当前不是在加载中
          if (this.needLoadMoreOnDidScroll  && this.currentState != State.Loading) {
            this.currentState = State.Loading
            //使用动画平滑的设置 footer 的偏移量为刚好完全露出它本身
            this.getUIContext().animateTo({
              duration: 500,
              curve: Curve.EaseOut,
              iterations: 1,
              playMode: PlayMode.Normal,
              onFinish: () => {
                console.error('加载更多时刚好漏出完整的 footer');
              }
            }, () => {
              this.footerTransitionY = -this.footerMaxTransition

            })


            this.footerText = '加载中'
            this.needLoadMoreOnDidScroll = false
            // 模拟加载更多网络延时
            setTimeout(() => {
              this.footerText = '加载成功'
              this.currentState = State.Idle
              this.lastLoadMoreTime = systemDateTime.getTime()
            //
              setTimeout(() => {
                 //500毫秒的动画，平滑的取消偏移量
                this.footerTransitionY = this.footerMaxTransition
                this.getUIContext().animateTo({
                  duration: 500,
                  curve: Curve.EaseOut,
                  iterations: 1,
                  playMode: PlayMode.Normal,
                  onFinish: () => {
                    console.error('加载更多后恢复 play end');
                    this.footerText = '上拉加载更多'
                  }
                }, () => {
                  this.footerTransitionY = 0

                })
              }, 1000)
            }, 2000)
          } else {
             // 这里对应着footer 没有完全漏出，松手后平滑的取消偏移量
            this.resetFooterTransition()
          }

        })
```

下面是两个取消偏移量的动画

```typescript
  resetHeaderTransition() {
    this.getUIContext().animateTo({
      duration: 500,
      curve: Curve.EaseOut,
      iterations: 1,
      playMode: PlayMode.Normal,
      onFinish: () => {
        console.error('刷新后恢复 play end');
      }
    }, () => {
      this.headerTransitionY = 0
    })

  }

  resetFooterTransition() {
    this.getUIContext().animateTo({
      duration: 500,
      curve: Curve.EaseOut,
      iterations: 1,
      playMode: PlayMode.Normal,
      onFinish: () => {
        console.error('刷新后恢复 play end');
      }
    }, () => {
      this.footerTransitionY = 0
    })
  }
```

这样我们就粗暴的实现了上拉加载更多和下拉刷新的组件。这里只是给出大致的思路，还有很多细节没有考虑：

- 比如嵌套滑动的情况
- 比如List内容不满一屏幕的情况等等

---

以上
