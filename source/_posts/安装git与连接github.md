---
title: 安装git与连接github
tags: [git,运维]
date: 2019-01-27 03:52:39
keywords: 安装git, 连接github
---

这可能是有史以来我写过的最啰嗦的博客了。
<!--more-->


#### 1. 安装github
##### 1.1  在linux上安装github
首先可以试着在shell中输入`git`，若提示没有该命令，则可以使用`sudo apt-get install git` 来安装，
##### 1.2  在windows上安装github
可以在[https://git-scm.com/downloads]https://git-scm.com/downloads) 下载，然后默认安装就可以了，安装后在空白处右键，会有`git GUI here`和`git bash here`。

#### 配置本地和远程git仓库

##### 2.1  配置本地环境

打开 git shell ,配置用户名和邮箱(这里的用户名邮箱与下文中的连接github或者其他三方git服务没有任何关系)    

```
git config --global user.name "Your Name"
git config --global user.email "email@example.com"
```

#####2.2  配置远程仓库
以github为例，其他git服务也类似
申请好账号之后，点击右上角自己的头像在弹出菜单中选择`settings`，看起来像下面这样：
![githubHomePage](/image/git/githubHomePage.png)
在新界面中的左侧列表中选择`SSH keys`，点击右上角的 `Add SSH key`，看起来像下面这样
![githubSSHkey](/image/git/githubSSHKEY.png)
然后我们回到shell中，继续生成密钥等工作
github官网有详细教程，链接地址请点[https://help.github.com/articles/generating-ssh-keys/#platform-windows](https://help.github.com/articles/generating-ssh-keys/#platform-windows) ，github网站访问速度慢的可以继续看下去，去github官网看的请跳过本章

##### 2.3  生成ssh keys
首先检测有没有已经生成好的ssh key，windows用户可以查看 `C:\Users\username\.ssh`文件夹下有没有`id_rsa.pub`文件，linux用户可以在shell中输入`ls -al ~/.ssh`，若有，请跳过本步骤。
在git shell 中输入下面第一条命令，linux用户注意不用加 `sudo`
``` shell 
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
Creates a new ssh key, using the provided email as a label
Generating public/private rsa key pair.
```
建议使用默认位置，直接回车就好，接下来会提示是否需要密码，不需要的话直接回车确认，需要的话输入密码，然后确认密码，然后会提示你密钥已经生成。
##### 2.5 将ssh key添加到你的github账户中
回到我们打开的添加SSHKEY网页，点击`Add SSH Key`，在`Title`中添加说明，用记事本之类的编辑器打开生成的`id_rsa.pub`（位于.ssh文件中），复制其中的内容到`Key`中，然后点击`Add Key`
##### 2.6 测试连接
打开 git shell 输入`ssh -T git@github.com`，若提示
``` shell
The authenticity of host 'github.com (207.97.227.239)' can't be established.
# RSA key fingerprint is 16:27:ac:a5:76:28:2d:36:63:1b:56:4d:eb:df:a6:48.
# Are you sure you want to continue connecting (yes/no)?
输入`yes`，如果你看到了如下提示
Hi username! You've successfully authenticated, but GitHub does not
# provide shell access.
``` 
说明连接成功，若有问题，请点击[这里](https://help.github.com/categories/ssh/).

----
以上
