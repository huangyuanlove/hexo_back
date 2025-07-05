---
title: 鸿蒙-JsonToArkTS插件更新
tags: [鸿蒙]

date: 2025-07-05 16:50:32
keywords: HarmonyOS,JsonToArkTS,json转数据类,json转model
---
## JsonToArkTs

鸿蒙应用状态管理V2已经移除了GAP说明，目前新开发的功能都是使用V2版本的状态管理，也有一部分已经完成的功能准备迁移到 V2.  
但在开发过程中，使用`@ObservedV2`装饰器时需要对每一个在UI中用到的属性加上`@Trace`装饰器，这纯纯的体力活，所以写了这个插件来自动生成这些代码。

## 使用方法

目前没有上线到应用市场，需要自行下载 jar 包安装。  
安装文件在 release 文件夹下，下载后在 IDEA 中选择`File`->`Settings`->`Plugins`->`Install Plugin from Disk`选择下载的 jar 包即可安装。
![安装插件](image/install_from_disk.png)

当前将该插件菜单放在了generate菜单最后一项，点击后即可出现操作弹窗
![JsonToArkTS](image/JsonToArkTS.png)

### 格式化

点击`Format`按钮即可对编辑区域内的字符串进行格式化，方便查看。直接高亮，折叠等操作
如果字符串不合法，在 IDE 底部会有对应提示
![错误提示](image/error_notification.png)

### 生成代码

输入区域粘贴我们需要进行转换的json字符串，点击下方 Generate 即可在当前文件中生成对应的数据类代码。  
默认类名为当前文件名，生成代码后会自动修改首字母大写的驼峰样式。  
默认是会为每个类属性都加上`@Trace`装饰器，如果不需要可以取消`with @Trace`选项。  
默认类属性是可空的，也就是说生成的代码会是这种样式。 
``` ts
@ObservedV2
export class Person{
  @Trace address?:Address
  @Trace name?:string
  @Trace age?:number
}

@ObservedV2
export class Address{
  @Trace zipCode?:number
  @Trace location?:string
}
```

如果不需要可空属性，可以选择`with default value`选项，生成的代码会是这种样式。
``` ts
@ObservedV2
export class Person{
  @Trace address:Address = new Address()
  @Trace name:string = ""
  @Trace age:number = 0
}

@ObservedV2
export class Address{
  @Trace zipCode:number = 0
  @Trace location:string = ""
}
```

### 生成解析方法：fromJSON和fromObject
勾选了`with fromJSON`后会生成对应的解析方法
``` ts
@ObservedV2
export class Address {
  @Trace zipCode: number = 0
  @Trace location: string = ""

  fromJSON(jsonStr: string): Address {
    let json: Address = JSON.parse(jsonStr) as Address
    return this.fromObject(json)
  }

  fromObject(obj: Address): Address {
    let tmp: Address = new Address()
    if (obj) {
      if (obj.zipCode != undefined) {
        tmp.zipCode = obj.zipCode;
      }
      if (obj.location != undefined) {
        tmp.location = obj.location;
      }
    }
    return tmp
  }
}

@ObservedV2
export class Person {
  @Trace address: Address = new Address()
  @Trace name: string = ""
  @Trace age: number = 0

  fromJSON(jsonStr: string): Person {
    let json: Person = JSON.parse(jsonStr) as Person
    return this.fromObject(json)
  }

  fromObject(obj: Person): Person {
    let tmp: Person = new Person()
    if (obj) {
      if (obj.address != undefined) {
        tmp.address = new Address().fromObject(obj.address);
      }
      if (obj.name != undefined) {
        tmp.name = obj.name;
      }
      if (obj.age != undefined) {
        tmp.age = obj.age;
      }
    }
    return tmp
  }
}

```
其中`fromJSON`调用了`fromObject`方法，可以认为是一种粗暴的深拷贝。  
这么做的原因是：ArkTS中想要使得观测能力生效，需要使用`new`关键字创建对象。

## 需要注意的地方

### 列表中的元素属性会被合并到同一个类中
例如我们有如下 json 字符串
```json
{
  "name": "Huang",
  "age": 22,
  "order_list": [
    {
      "pay_method": "wx",
      "time": "2021-01-01 11:12:21",
      "id": "13323"
    },
    {
      "pay_method": "wx",
      "marked": "周末配送"
    }
  ]
}
```
在`order_list`列表中，第一项有id属性，第二项有marked属性，这两个属性会被合并到同一个类中，生成的代码如下
``` ts
@ObservedV2
export class FailedClassArticle{
  @Trace name:string = ""
  @Trace order_list:OrderList[] = []
  @Trace age:number = 0
}

@ObservedV2
export class OrderList{
  @Trace marked:string = ""
  @Trace pay_method:string = ""
  @Trace time:string = ""
  @Trace id:string = ""
}
```
### 不支持列表项为不同的类型
例如如下 json
```json
{
  "order_list": [
    1,
    2,
    3,
    {
      "pay_method": "wx",
      "time": "2021-01-01 11:12:21",
      "id": "13323"
    },
    {
      "pay_method": "wx",
      "marked": "周末配送"
    }
  ]
}
```
虽然这是一个合法的 JSON 字符串，但是由于列表中的元素类型不同，前三项为数字，后两项为对象，会导致生成的代码不符合预期。

