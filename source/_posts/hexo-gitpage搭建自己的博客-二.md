---
title: hexo+gitpage搭建自己的博客(二)
date: 2016-10-25 21:11:53
tags: [hexo,gitpage,blog]
keywords: hexo,gitpage,blog
---

之前我们已经安装好了`hexo`，接下来就是初始化了。
```shell
    hexo init <dir>
    cd <dir>
    npm install
```
<!-- more -->
![hexo_install](/image/hexo/hexo_init.png )
![hexo_install](/image/hexo/npm_install.png)
现在hexo就安装完成了，在存放hexo的文件夹目录下执行 `hexo s`，就可以启动hexo的服务，启动之后有提示，在浏览器中输入`127.0.0.1:4000`就可以看到最初的效果了，如下
![hello_world](/image/hexo/hexo_hello.png)
***
　　对了，还要发布到github。前提准备是在上一篇中已经创建好了github的仓库。接下来在存放hexo资源的文件夹下(以下用 heox_blog这个文件夹代替)执行
`npm install hexo-deployer-git --save`
修改hexo_blog文件夹下的`_config.yml`文件
在末尾添加
```shell
deploy:
  type: git
  repository: git@github.com:your name/your name.github.io.git
  branch: master
```
注意，type，repository，branch冒号后面都有一个空格。保存后执行
```hexo
    hexo clean
    hexo g
    hexo d
```
就可以将写好的博客部署到github上了。（部署到github时建议按照顺序执行以上命令）
***
```hexo
hexo generate == hexo g    -->将md文件解析成静态的html文件
hexo deploy == hexo d      --> 将文件部署到github
hexo server == hexo s      --> 启动本地hexo服务
hexo clean				   --> 清除缓存
hexo new "title" 			--> 创建新的文章，文件在`hexo_blog/source/_posts`文件夹下
```
***
以上
