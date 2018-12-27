---
title: Ubuntu输入密码登陆后回到登录界面
tags: [linux]
photos:
  - /image/linux/update_ubuntu_18.04.jpeg
date: 2018-12-26 11:25:57
keywords: [ubuntu无法开启桌面,ubuntu登录成功后返回登录界面,ubuntu登录问题]
---

既然推荐更新了，那就更呗，然后更新到了18.04，被小伙伴吐槽不用命令行更新的系统是没有灵魂的。更新之后玩了没几天，昨晚突然间无法登录桌面了。输入密码登录后显示一下桌面背景然后就回到了的登录界面，使用tty登录没问题。这就好办了，盲猜桌面服务挂了。然后猜原因：

1. `.Xauthority`权限不对
2. `/tmp`权限不对
3. `ubuntu-desktop`挂了

盲猜完了，验证一下

<!--more-->

#### .Xauthority 权限不对

`Xauthority`是`startx`脚本记录文件。`Xserver`启动时，读文件`~/.Xauthority`，读入相应其`display`的记录。

当一个须要显示的客户程序启动调用`XOpenDisplay()`也读这个文 件。并把找到的`magic code` 发送给`Xserver`。

当`Xserver`验证这个`magic code`正确以后，就允许连接啦。

观察`startx`脚本也能够看到，每次`startx`执行，都在调用`xinit`曾经使用了`xauth`的`add`命令加入了一个新的记录到`~/.Xauthority`，用来这次执行X使用认证 。

查看文件权限，没问题，查看`/tmp`权限没问题。

emmmmmmmmmmmmmmmmmmmmmmmmmmmmmmmmmmmm

家目录下有个`.xsession-errors`文件，也没有错误日志。。。

#### ubuntu-desktop 挂了

tty登录，删除`ubuntu-desktop`

> sudo apt-get autoremove ubuntu-desktop

然后 `ctrl+alt+f1`进入GUI，输入密码登录。成功。

emmmmmmmmmmmmmmmmmmmmmmmmmmmmmmmmm

只是桌面上的文件不显示了，不急，命令行查看一下文件还在，放心了。

dock(launch)不见了，就是在桌面左边放软件图标的快捷启动，相当于windows的任务栏的那玩意

按下win键，搜索还能用，问题不大。

原来安装的软件还能用，那就更没问题了。

> sudo apt-get install desktop-base
> sudo apt-get install desktop-profiles
> sudo apt-get install desklaunch
> sudo apt-get install gnome-shell-extension-ubuntu-dock

ok，解决

没必要追求所有软件和依赖都使用最新版的，有时间折腾还好，没时间折腾。。。。没时间折腾的就要么老老实实用windows，要么上mac。

----

操作系统：我们要不要信任用户的操作？

windows：不信任，用户都是傻~~消音~~，不信任，只开放部分权限给用户，系统配置更新我们自己搞。

ubuntu：用户安装了操作系统，那就所有东西都放权给用户，我们只在关键地方给出提示。



----

以上