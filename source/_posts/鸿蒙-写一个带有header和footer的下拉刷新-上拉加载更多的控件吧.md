---
title: 鸿蒙-写一个带有header和footer的下拉刷新,上拉加载更多的控件吧-Refresh组件
tags: [HarmonyOS]
date: 2026-01-20 16:27:03
keywords: HarmonyOS,下拉刷新,上拉加载更多,pullToRefresh,
---

## 前言

页面有个列表需要加一下下拉刷新和上拉分页加载的功能，这不很简单嘛，下拉刷新套一层`Refresh`，上拉加载更多就在 List的尾部添加一个展示加载中的 ListItem,数据返回后判断有没有更多数据，有就展示，没有就不展示。
完活。
后面想想可以自己做一下类似 Android 上的那种 PullToRefresh 的效果。在网上翻了翻，哎嘿还真有，嗯，那就自己写一写试一试

## 自带组件-Refresh

这个就比较简单了， 按照官方文档来搞就行。但大部分情况会对刷新区域的内容有些要求，Refresh 组件也有自定义刷新区域的参数。

### 最简单的

先来看下最简单的方案

```typescript
import { ActionBar } from '../../../comm/ActionBar';

const numbers: number[] = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20];
@Entry
@ComponentV2
struct ListRefreshPage {
  @Local isRefreshing:boolean = false
  @Local data:number[]=numbers
  @Local refreshText:string ='继续下拉进行刷新'

  build() {
    Column() {
      ActionBar({title:"下拉刷新和上拉加载更多"})
      Refresh({refreshing:$$this.isRefreshing,promptText:this.refreshText}){
        List({space:20}){
          ForEach(this.data,(item:number,index:number)=>{
            ListItem(){
              Text(`第 ${item+1} 个`).height(50)
            }
          })
        }
      }.onStateChange((refreshStatus: RefreshStatus) => {
        if(refreshStatus == RefreshStatus.Drag){
          this.refreshText = '继续下拉进行刷新'
        }else if(refreshStatus == RefreshStatus.OverDrag){
          this.refreshText = '松开刷新'
        }else if(refreshStatus == RefreshStatus.Refresh){
          this.refreshText = '刷新中'
        }
      })
      .onRefreshing(() => {
        setTimeout(() => {
          this.isRefreshing = false;
        }, 2000)

      })
      .layoutWeight(1).width("100%")
    }
    .height('100%')
    .width('100%')
  }
}
```

我们需要在下拉刷新的过程中，提示用户下拉到什么程度松手就可以刷新了。这里我们可以传入`promptText`,在`onStateChange`的回调中判断当前状态，进而更新文案，达到提示用户的目的，效果如下：
![更新下拉刷新文案](image/HarmonyOS/refresh_component.gif)

### 自定义刷新头部组件

这个也比较简单，给了两种写法，一种是自定义builder，另外一种是自定义refreshingContent

#### 自定义 builder

```typescript
  @Builder
  customRefreshComponent() {
    Stack() {
      Row() {
        LoadingProgress().height(32)
        Text("Refreshing...").fontSize(16).margin({ left: 20 })
      }
      .alignItems(VerticalAlign.Center)
    }
    .align(Alignment.Center)
    .clip(true)
    // 设置最小高度约束保证自定义组件高度随刷新区域高度变化时自定义组件高度不会低于minHeight。
    .constraintSize({ minHeight: 32 })
    .width("100%")
  }

  Refresh({ refreshing: $$this.isRefreshing, builder: this.customRefreshComponent() }) {}
```

这里是传入了 builder 参数。

#### 自定义refreshingContent

这个相对复杂一些，因为 Refresh 的refreshingContent参数需要一个ComponentContent对象。

```typescript
import { ComponentContent } from '@kit.ArkUI';

class Params {
  refreshStatus: RefreshStatus = RefreshStatus.Inactive;

  constructor(refreshStatus: RefreshStatus) {
    this.refreshStatus = refreshStatus;
  }
}

@Builder
function customRefreshingContent(params: Params) {
  Stack() {
    Row() {
      LoadingProgress().height(32)
      Text("refreshStatus: " + params.refreshStatus).fontSize(16).margin({ left: 20 })
    }
    .alignItems(VerticalAlign.Center)
  }
  .align(Alignment.Center)
  .clip(true)
  // 设置最小高度约束保证自定义组件高度随刷新区域高度变化时自定义组件高度不会低于minHeight。
  .constraintSize({ minHeight: 32 })
  .width("100%")
}

```

然后我们在组件的aboutToAppear方法中构建一下ComponentContent对象

```typescript
private contentNode?: ComponentContent<Object> = undefined;
private params: Params = new Params(RefreshStatus.Inactive);

aboutToAppear(): void {
  let uiContext = this.getUIContext();
  this.contentNode = new ComponentContent(uiContext, wrapBuilder(customRefreshingContent), this.params);
}
```

使用的时候

```typescript
Refresh({
  refreshing: $$this.isRefreshing,
  refreshingContent: this.contentNode,
});
```

## 加载更多

需要定义一个变量来控制是否展示最后一个`加载更多`的 ListItem

```typescript
//获取到数据之后，根据约定的字段判断是否展示加载更多
@Local hasMore:boolean = true
@Local isLoading:boolean = false
...
List({space:20}){
  ForEach(this.data,(item:number,index:number)=>{
    ListItem(){
      Text(`第 ${item+1} 个`).height(50)
    }
  })
  if(this.hasMore){
    ListItem(){
      Row(){
        Text('正在加载')
      }
    }
  }
}.onScrollIndex((start: number, end: number, center: number) =>{
    if(end >= this.data.length -1 && !this.isLoading){
      this.isLoading =true
      setTimeout(()=>{
        this.isLoading =false
        this.data.push(...numbers)
        this.hasMore = false
      },2000)
    }
  })

```

在`onScrollIndex`中，判断是否滑动到了底部还有其他方法，但或多或少都有些问题。
比如有一个`onReachEnd(event: () => void)`回调，但是它在设置edgeEffect属性为EdgeEffect.Spring时，每次滑动到底部都会回调两次。
还有可以给 List 指定一个 scroller 属性，利用`scroller.isAtEnd()`判断是否滑动到了底部，但是我们找不到一个合适判定时机。

---

上面就是最简单的下拉刷新，上拉加载更多的写法了。
下一篇我们看看不使用 Refresh 和添加 ListItem 的方式，如何实现这种效果
