---
title: 使用express模拟后台返回数据
date: 2018-09-27 14:52:19
tags: [Android,]
keywords: 模拟后端,模拟接口数据,express
---
在研发过程中，有时候会遇到前端写完了，但是后端接口还没有完成的情况。一般情况下我们会写一些假数据来填充UI，这种方式没有办法检测网络请求有没有问题。我们可以自己搭一个服务，请求自己的服务来返回一些模拟数据。比如可以使用node和express模块来做。
<!--more-->
#### 安装node
官网自己查
安装node会自动安装npm包管理工具，检查是否有安装node和npm包管理工具，可以通过执行`node -v` 和 `npm -v`来查看，如果已经安装则会输出对应的版本号
![node和npm](/image/Android/express/node_and_npm.png)

#### 安装express
官网在这里 http://expressjs.com/zh-cn/
1. 全局安装 `express-generator`
   ``` shell
    npm install -g express-generator
   ```
2. 创建一个测试工程
   ``` shell
    express --view=pug testExpress
   ```
成功后会自动在目标位置创建一个名为myapp的项目并生成很多文件
![创建myapp](/image/Android/express/create_myapp.png)
然后根据提示执行
``` shell
cd testExpress && npm install
``` 
之后执行 `npm start`
提示 
``` shell
> myapp@0.0.0 start /Users/huangyuan/myapp
> node ./bin/www
```
表示启动服务成功，然后在浏览器输入`http://127.0.0.1:3000`,界面显示`Welcome to Express`说明服务已经启动成功

#### 项目结构
安装完成后整个工程目录结构如下：
![工程结构](/image/Android/express/myapp_constructor.png)
既然是模拟后台的返回数据，我们只需要关注`app.js`文件和`routers`文件夹就可以了。
在`routers`文件夹中有`index.js`和`user.js`文件，这两个文件在`app.js`文件中注册并使用：
``` js
var indexRouter = require('./routes/index');
var usersRouter = require('./routes/users');
app.use('/', indexRouter);
app.use('/users', usersRouter);
```
意思差不多就是访问根目录，则返回`index.js`文件中`router.get`注册为`/`的方法中`res.render`的值。
同样的，我们访问`/users`则会返回该文件中`data`字段的值。

#### 仿写
仿造上面两个示例，我们可以写点其他的东西。
在`routes`文件夹下新建`verson.js`,内如如下：

``` javaScript
var express = require('express');
var router = express.Router();
var data = {
  'code':0,
  'message':'success',
  'version':{
      "versionCode":"1.0.1",
      "versionName":"机遇"
  }
}
router.get('/', function(req, res, next) {
  res.send(data);
});
module.exports = router;
```

在`app.js`中注册：
``` javaScript
var versionRouter = require('./routes/version')
...
app.use('/versioninfo',versionRouter)
```
需要注意的是在`app.use`方法中传入的第一个参数就是我们要访问的路径
重启服务，然后访问一下，如果返回了正确的json数据，说明配置成功。在修改配置或者添加内容的时候，如果再次访问没有成功或者还是原来的内容，注意看一下控制台是不是被304缓存了。

这样我们只需要在研发阶段把地址指向我们自己的服务，联调时指向测试服务器就可以了。

------
以上
