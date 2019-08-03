---
title: 树莓派安装owncloud
tags: [树莓派,linux]
photos:
  - /image/raspberry_pi/ubuntu_server_4_raspberry_pi_.png
date: 2019-08-03 22:45:08
keywords: [raspberryPi,树莓派,owncloud]
---

买了个新玩具：树莓派3b+，挺好玩。

1. 一个开发板(包含cpu、内存、网卡、usb接口)。
2. 一张扩展卡(内存卡，就是之前手机上用的那种TF拓展卡，用来装系统，开发板上有专门的卡槽用来安装扩展卡)。
3. 一根网线(其实开发板上是有无线网卡的，据说配置起来听麻烦，就用网线上，插上就能用)
4. 一根HDMI视频线，毕竟安装系统的时候可能会用到
5. 其他配件，比如散热风扇、外壳、摄像头什么的随意，毕竟不是必须的

<!--more-->

### 安装系统

1. 系统： 可以在https://www.raspberrypi.org/downloads/ 查看，我是习惯了ubuntu，下载的是 https://ubuntu.com/download/iot/raspberry-pi-2-3 这个系统，
2. 需要的软件：专门的格式化工具SDFormatter格式化内存卡；windows系统下安装烧写镜像的工具：Win32DiskImager
3. 将下载的系统写到tf卡上，将卡插入开发版，通电，启动的比较慢，可以在屏幕上看到一些输出，根据自己的需要配置一下就好。

### 安装 owncloud

#### 所需环境

1. php环境及必须的组件

> 安装Apache 网页服务器
>
> **$ sudo apt-get install apache2**

> 安装Mysql 数据库
>
> **$ sudo apt-get install mysql-server mysql-client**

> 安装PHP
>
> **$ sudo apt-get install php7.2-mysql php7.2-curl php7.2-json php7.2-cgi libapache2-mod-php7.2 php7.2**
>
> **$ sudo apt-get install php7.2-gd php7.2-intl php7.2-imagick**

> 安装phpmyadmin并设置mysql的密码
>
> **$ sudo apt-get install phpmyadmin**

根据返回的消息提示进行操作就好，遇到问题可自行搜索解决，解决不了，可以退出了。

####  检查php环境是否可用

在 `/var/www/html/`下新建一个php文件，比如`test.php`，输入 

``` php
<? php
    phpinfo();
?>
```

保存一下，重启apache服务，`sudo /etc/init.d/apache2 restart`，访问 `http://ip/test.php`，如果能看到php版本，则说明环境没问题。如果看不到，请自行解决；解决不了，可以退出了。

#### 下载owncloud源码并部署

>  下载安装包。
>
>  **$ wget** **https://download.owncloud.org/community/owncloud-10.0.9.tar.bz2**
>
>  解压安装包
>
>  **$ tar -xvf owncloud-10.0.9.tar.bz2**
>
>  将所有解压后的文件移到 /var/www/html
>
>  **$ sudo mv owncloud/\*  /var/www/html**
>
>  修改Apache的配置文件apache2.conf：
>
>  **$ sudo nano /etc/apache2/apache2.conf**
>
>  向下查找到`AllowOverride`，将None修改为All
>
>  ``` xml
>  <Directory /var/www/>
>  	Options Indexes FollowSysmLinks
>  	AllowOverride All
>  	Require all granted
>  </Directory>
>  ```
>
>  创建data文件夹，用于保存数据
>
>  **$cd /var/www/html**
>
>  **$sudo mkdir data**
>
>  修改Owncloud文件夹的文件权限：
>
>  **$sudo chown -R www-data:www-data /var/www/html/**
>
>  **$sudo chmod 777 /var/www/html/config/**
>
>  在MySql上创建一个数据库，保存来自OwnCloud的数据。
>
>  ``` mysql
>  $ sudo mysql -u root -p
>  create database owncloud;
>  GRANT ALL ON owncloud.*TO username@localhost IDENTIFIED BY 'password';
>  flush privileges;
>  exit
>  ```
>
>  重启Apache服务器
>
>  **$** **sudo /etc/init.d/apache2 restart**

#### 访问owncloud并配置

在浏览器上输入树莓派的IP地址

![owncloud index](/image/raspberry_pi/owncloud_page_config.png)

点击配置完成后，等待的时间可能会稍微长一点，可以另开一个tab访问一下试试。



#### 其他

1. 如果选择的存放数据的地方，是挂载到linux的nfts格式的硬盘，会出现无法修改其权限的问题，解决方案可以搜一下`linux无法修改ntfs权限`，网上解决方案大部分是修改 /etc/fstab来实现挂载时指定权限。我是把ntfs格式修改成ext4就好了。。。。。。

2. owncloud8之后，直接将文件放在用户的共享文件夹下，会发现在网页上并不能看到该文件。

   第一种是执行Owncloud的CLI命令。具体位置是在Owncloud根目录，有个文件叫OCC，实际调用的是console.php，是个PHP脚本。通过命令 PHP OCC fils:scan –all即可重新扫描文件目录下的所有文件了。代价是所有文件的ID会重置，导致同步的桌面客户端会认为都是新文件，继而全部重新下载。

   存放共享文件数据的数据库是`oc_filecache`表，我们可以先truncate掉这张表，之后重新登录OwnCloud，Owncloud系统会自动重新填充数据。



--------

以上




