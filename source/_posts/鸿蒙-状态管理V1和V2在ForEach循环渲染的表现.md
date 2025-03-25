---
title: 鸿蒙-状态管理V1和V2在ForEach循环渲染的表现
tags: [HarmonyOS]
date: 2025-03-24 22:11:33
keywords: HarmonyOS,状态管理,Foreach循环渲染
---
状态管理V2已经出来好长时间了，移除GAP说明也有一段时间了，相信有一部分朋友已经开始着手从V1迁移到V2了，应该也踩了不少坑。
下面向大家分享一下我使用状态管理V1和Foreach时遇到的坑，以及状态管理V2在Foreach循环渲染中的表现。

## 前提
这里就先默认大家都已经熟悉状态管理V1中的@Observed装饰器和@ObjectLink装饰器，以及ForEach循环渲染相关的知识，并且仔细阅读过`ForEach：循环渲染`章节中的[渲染结果非预期](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/arkts-rendering-control-foreach#渲染结果非预期)了。

## 遇到的问题

先说场景需求：
典型的支付结算页面选择优惠券的场景。当用户在结算页面点击优惠券时，跳转到优惠券列表页面，并在该页面向服务器请求优惠券列表数据。
这是服务器会根据传入的订单信息按照需求计算出默认选中哪个优惠券，该页面支持下拉刷新。
我们来简化一下优惠券数据，关键数据优惠券id，抵扣信息和描述。于是我们很容易写出如下代码：
``` TypeScript
//数据类
@Observed
class CouponData {
  id:string = ''
  name: string = ''
  defaultSelect: boolean = false
}

//用于展示数据的控件
@Component
struct CouponView {
  @Watch('onCouponDataChange') @ObjectLink model: CouponData
  //优惠券是单选，因此选中|取消选中优惠券时通知父组件更新数据
  onChangeSelect:(id:string,select:boolean)=>void = (id:string,select:boolean)=>{}

  onCouponDataChange() {
    hilog.error(0x01, 'ForeachPage', `onCouponDataChange ${this.model.id}  ${this.model.defaultSelect}`)
  }

  build() {
    Row() {
      Text(`${this.model.name} , select :${this.model.defaultSelect}`)
      Circle()
        .width(20)
        .height(20)
        .fill(this.model.defaultSelect ? Color.Red : Color.Gray)
        .stroke(this.model.defaultSelect ? Color.Red : Color.Grey)

    }.padding({ top: 10, bottom: 10 }).onClick((_)=>{
        this.onChangeSelect(this.model.id,!this.model.defaultSelect)
    })
  }
}

//为了简单展示，这里没有从服务器获取数据；下拉刷新也用按钮代替；点击确认时弹个toast提示一下选中的优惠券id



@Entry
@Component
struct ForeachPage {
  @State couponDataList: CouponData[] = []
  aboutToAppear(): void {
    this.initData()
  }
  //模拟一下数据
  initData() {
    this.couponDataList = []
    for (let i = 0; i < 5; i++) {
      let model: CouponData = new CouponData()
      model.id= i.toString()
      model.name = `优惠券 ${i} 项`
      if (i == 1) {
        model.defaultSelect = true
      } else {
        model.defaultSelect = false
      }
      this.couponDataList.push(model)
    }
  }

  build() {
    Column() {

    //就当这里是下拉刷新了，问题不大
      Button("刷新").onClick((_)=>{
        this.initData()
      })
      List() {
        ForEach(this.couponDataList, (model: CouponData) => {
          ListItem() {
            CouponView({ model: model ,onChangeSelect:(id:string,select:boolean)=>{
              hilog.error(0x01, 'ForeachPage', `onChangeSelect ${id} ${select}`)
              this.couponDataList.forEach((data:CouponData)=>{
                if(data.id == id){
                  data.defaultSelect = select
                }else{
                  if(select){
                    data.defaultSelect =false;
                  }
                }
              })
            }})
          }

        }, (item: CouponData,index:number) => {
          let key = item.id +"__" +item.defaultSelect
          hilog.error(0x01, 'ForeachPage', key)
          return key
        })
      }.layoutWeight(1)

      Button("确定").onClick((_)=>{
        let selectCouponID:string = '未选中';
        this.couponDataList.forEach((couponData:CouponData)=>{
          if(couponData.defaultSelect){
            selectCouponID = couponData.id
          }
        })
        promptAction.showToast({message:`选中的优惠券是 ${selectCouponID}`})
      })

    }
    .height('100%')
    .width('100%')
  }
}

```

使用了ForEach循环渲染来生成List的子组件，并且根据开发文档的[使用建议](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/arkts-rendering-control-foreach#使用建议)，我们没有让index参与key的生成，而是使用优惠券的唯一id作为key。
运行后切换选中状态，完美。  
但是遇到了两个问题：
1. 点击刷新后，并没有将第二项设置未选中、其他项设置为未选中。  
2. 没有办法切换选中状态。  
-----emmmmmm------  
不急，肯定有它的原因。  

看日志：发现在切换选中状态的时候列表项的key没有打印，说明选中状态的切换也就是UI的刷新不是因为key发生了变化，而是因为ObjectLink和Observed观测能力驱动的UI发生的变化。

接着就能确认问题1：因为切换选中状态时key没有变化，导致点击刷新之后，第二次列表的key和刚进入时列表key一致，因此UI没有刷新。
但这里有个问题：为什么参与计算key的属性发生了变化，但key却不会变化？这可能和ObjectLink和Observed观测能力的实现有关，这里没有确认。

但为什么没有办法切换选中状态？看文档中@State是可以观测到数组项赋值的。
根据问题1的结论接着推论：因为key相同，不会重新绘制列表项，这就引起了另外一个问题：列表项没有被重新绘制，因此列表项还是绑定着点击刷新之前数组中的对象，但我们点击列表项时，修改的是数组中的新对象，因此更不会刷新UI。

为了验证这个推论，我们第一次对数组赋值时将第二项默认选中设置为true； 点击刷新的时候，将第四项默认选中设置为true。
修改一下initData方法
``` TypeScript
    firstInit:boolean = true;
  initData() {
    this.couponDataList=[]
    for (let i = 0; i < 5; i++) {
      let model: CouponData = new CouponData()
      model.id= i.toString()
      model.name = `优惠券 ${i+1} 项`

      if(this.firstInit){
        if (i == 1 ) {
          model.defaultSelect = true
        } else {
          model.defaultSelect = false
        }
      }else{
        if (i == 3 ) {
          model.defaultSelect = true
        } else {
          model.defaultSelect = false
        }
      }
      this.couponDataList.push(model)
    }
    this.firstInit = false;
  }
```

这时候，我们进入页面，默认选中了第二项。然后点击第一项，将第一项切换为选中状态。之后点击刷新。发现第一项和第四项都变成了选中状态。

这时候我们点击第二项，可以将第二项切换为选中状态，并且第四项切换为未选中状态。这是是因key发生了变化，列表项重绘，绑定了数组中新的对象。

然后点击第三项或者第五项，都可以将第二项切换为未选中状态，但第三项和第五项本身不会被选中。因为第三项和第五项没有重绘，还是绑定的数组中之前的对象。

这时候选中第二项或者第四项之后，再点击第一项，发现并没有将第二项或者第四项切换为未选中状态，这是因为第一项没有被重绘，绑定的还是数组中之前的对象，并且是选中状态，这时候我们点击第一项是取消第一项的选中，并不会修改其他数据。

这里也验证了我们上面的推论。

这里就有人问了：
>emmm，那怎么办？
>凉拌呗，换V2。
>不行哇，这个数据类在其他地方也在用，还都是用的V1。
>你看，着kpi不就有着落了嘛

好吧，也有个比较恶心的办法，不追求极致性能、数据量较小的时候可以拿来应急：
定义一个变量，让这个变量参与key的生成，并且在每次刷新的时候都修改这个变量，进而达到强制让key发生变化，重绘所有列表项。

``` TypeScript
refreshTime:number = 0;
initData() {
    this.refreshTime = systemDateTime.getTime()
    ...
}
//ForEach额key生成方法
(item: CouponData,index:number) => {
          let key = item.id +"__" +item.defaultSelect +"__"+this.refreshTime
          hilog.error(0x01, 'ForeachPage', key)
          return key
        }
```
emmm，这样可以正常刷新。

## 换V2呗

改动也没多少，不过有一点比较恶心，就是被@ObservedV2修饰的类，参与UI展示的属性必须被@Trace修饰，属性少了还好说，属性多了纯纯体力活。
写了个插件，可以从json字符串转为ArkTS对象，并且自动加上@Trace修饰
[github](https://github.com/huangyuanlove/JsonToArkTS)  
[gitee](https://gitee.com/huangyuan/JsonToArkTS)   
[gitcode](https://gitcode.com/huangyuan_xuan/JsonToArkTS)   



``` TypeScript
@Entry
@ComponentV2 //修改为V2
struct ForeachPage {
  @Local couponDataList: CouponData[] = [] //修改为V2

  aboutToAppear(): void {
    this.initData()
  }

  initData() {

    this.couponDataList = []
    for (let i = 0; i < 5; i++) {
      let model: CouponData = new CouponData()
      model.id = i.toString()
      model.name = `优惠券 ${i + 1} 项`


      if (i == 1) {
        model.defaultSelect = true
      } else {
        model.defaultSelect = false
      }

      this.couponDataList.push(model)
    }

  }

  build() {
    Column() {
      Button("刷新").onClick((_) => {
        this.initData()
      })
      List() {
        ForEach(this.couponDataList, (model: CouponData) => {
          ListItem() {
            CouponView({
              model: model, onChangeSelect: (id: string, select: boolean) => {
                hilog.error(0x01, 'ForeachPage', `onChangeSelect ${id} ${select}`)
                this.couponDataList.forEach((data: CouponData) => {
                  if (data.id == id) {
                    data.defaultSelect = select
                  } else {
                    if (select) {
                      data.defaultSelect = false;
                    }
                  }
                })
              }
            })
          }

        }, (item: CouponData, index: number) => {
          let key = item.id + "__" + item.defaultSelect
          hilog.error(0x01, 'ForeachPage', key)
          return key
        })
      }.layoutWeight(1)

      Button("确定").onClick((_) => {
        let selectCouponID: string = '未选中';
        this.couponDataList.forEach((couponData: CouponData) => {
          if (couponData.defaultSelect) {
            selectCouponID = couponData.id
          }
        })
        promptAction.showToast({ message: `选中的优惠券是 ${selectCouponID}` })
      })

    }
    .height('100%')
    .width('100%')
  }
}


@ComponentV2   //修改为V2
struct CouponView {
  @Require @Param model: CouponData   //修改为V2
  @Event //修改为V2
  onChangeSelect: (id: string, select: boolean) => void = (id: string, select: boolean) => {
  }
  aboutToAppear(): void {
    hilog.error(0x01, 'ForeachPage', `aboutToAppear ${this.model.id}`)
  }

  build() {
    Row() {
      Text(`${this.model.name} , select :${this.model.defaultSelect}`)
      Circle()
        .width(20)
        .height(20)
        .fill(this.model.defaultSelect ? Color.Red : Color.Gray)
        .stroke(this.model.defaultSelect ? Color.Red : Color.Grey)

    }.padding({ top: 10, bottom: 10 }).onClick((_) => {
      this.onChangeSelect(this.model.id, !this.model.defaultSelect)

    })
  }
}

@ObservedV2 //修改为V2
class CouponData {
  id: string = ''
  @Trace name: string = '' //修改为V2
  @Trace defaultSelect: boolean = false //修改为V2
}

```

当我们切换选中状态，然后点击刷新后，再次切换选中状态也是正常的。通过`CouponView`中`aboutToAppear`方法的日志，也可以看到只重绘了key发生改变的列表项。

所以，那么，因此，迁移到V2不？

你问我迁移了吗？正在迁移，或许等到V3出来，我就迁移完了。