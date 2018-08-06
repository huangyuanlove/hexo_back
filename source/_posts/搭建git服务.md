---
title: 搭建git服务
date: 2017-04-15 10:25:31
tags: [git,运维]
keywords: git服务
---
公司的版本控制要从SVN迁移到git，正式的开发环境还没有搭建好，于是自己做了一个简单git服务。
环境：
本机: win10，服务器：ubuntu 16.04 LTS,ip:192.168.1.103
<!--more-->
本地安装git环境，配置用户名和邮箱之类的信息，然后生成秘钥，生成方式见 http://blog.csdn.net/huangyuan_xuan/article/details/49125597。
服务器：
1. 安装git,ssh服务
`sudo apt install git ssh`
2. 新增用户，用户名为git
`adduser git`
3. 初始化git仓库，我放在/home/git/repository
`cd /home/git/repository`
`git init --bare test.git`
`--base`参数是初始化裸仓库。
执行`tree`命令可以查看目录结构如下：
![git_init_tree](/image/git/git_init_tree.png) 
4. 添加秘钥
将本地生成的`id_rsa.pub`文件里面的内容追加到`/home/git/.ssh/authorized_keys`文件中。
可以先将秘钥文件上传到服务器，然后在服务器上操作文件，添加内容。
5. 修改权限
将 /home/git 所有者更改为git用户
`chown -R git:git /home/git`
用户home目录755权限
`chmod 755 /home/git`
.ssh目录700权限
`chmod 700 .ssh`
authorized_keys 600权限
`chmod 600 .ssh/authorized_keys`
6. 修改ssh配置文件
配置文件是`/etc/ssh/sshd_config`，取消这行 `AuthorizedKeysFile    %h/.ssh/authorized_keys` 前面的注释
7. 重启`ssh`服务
`sudo service ssh restart`
8. 可选项：为了安全 禁止git用户shell登录，需要修改/etc/passwd
将 git:x:1001:1001:,,,:/home/git:/bin/bash
改为 git:x:1001:1001:,,,:/home/git:/usr/bin/git-shell

回到本地，进行克隆
`git clone git@192.168.1.103:/home/git/code/test.git`
或者
`git clone git@192.168.1.103:code/test.git`
如果ssh不是默认的22端口，则在ip后添加端口。