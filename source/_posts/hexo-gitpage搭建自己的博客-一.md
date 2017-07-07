---
title: hexo+gitpage搭建自己的博客(一)
date: 2016-10-21 00:04:26
tags: [hexo,gitpage,blog]
---
**不说环境直接写配置的行为都是耍流氓**
按照惯例,先说自己的环境:`ubunu`,然后就没有然后了.<hr>
<!-- more -->
　　`hexo`类似`wordpress`的博客框架,`gitpage`是`github`的一个服务,请原谅我这个不大准确的表达,事实上真的差不多.
安装`hexo`需要安装`nodejs`,使用`gitpage`需要`git`,至于如何安装`git`,在原来的csdn博客上有,[点击这里](http://blog.csdn.net/huangyuan_xuan/article/details/49125597) http://blog.csdn.net/huangyuan_xuan/article/details/49125597.
之后需要在自己的github上创建一个仓库,名称结构如下:`username.github.io`,例如我的github主页是`https://github.com/huangyuanlove`,那么我的`gitpage`就是`huangyuanlove.github.io`.
到这里,默认大家的gitpage和git已经配置好了,包括sshkey之类的东西.
***
　　我安装`node`是用`nvm`(node version manager)安装的,然后使用`node`中的`npm`(node package manager)安装hexo.windows下安装请转[这里](https://github.com/coreybutler/nvm-windows) https://github.com/coreybutler/nvm-windows
首先安装`nvm`:
> `wget -qO- https://raw.github.com/creationix/nvm/master/install.sh | sh  `

![安装nvm](/image/hexo/hexo_install_nvm.png  "安装nvm图片")
安装时间长短视网速而定,我安装了大概十多分钟,然后重启一下终端.
然后使用 `nvm ls-remote`查看一下有哪些本版可以安装,我当时安装的最新版是`6.6.0`,现在不知道是哪一版,如下
![nvm ls-remote](/image/hexo/nvm_ls-remote.png)
找一个合适的版本使用如下命令安装
`nvm install version`,例如 `nvm install 6.6.0`
![nvm ls-remote](/image/hexo/nvm_install_6.6.0.png),安装时间还是视网速而定,我也忘了装了多长时间了.
安装完成之后是这样的
![nvm_install_down](/image/hexo/nvm_install_down.png)
之后`npm install -g hexo`安装`hexo`,`-g`参数是全局安装
![npm_install_hexo](/image/hexo/npm_install_hexo.png)
安装完成之后使用`hexo -v`查看hexo的版本号
![hexo-v](/image/hexo/hexo-v.png)
到此`hexo`安装完成,接下来就是初始化,进行配置了.下一篇再说吧,睡觉了.
***
以上.
