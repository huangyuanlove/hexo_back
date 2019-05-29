---
title: ss、ssr和v2ray以及bbr
tags: [linux,运维]
date: 2019-05-29 10:11:59
keywords: ss,shadowsocks,bbr,v2ray
---

由于众所周知的原因，为了防止文章被和谐，文中会出现明显(脑残)的解释说明上的错误，懂的自然都懂

使用pip安装ssserve以及对应的配置

使用bbr拥塞控制算法进行加速

使用v2ray脚本安装v2ray

<!--more-->

#### 安装ssserver及配置

##### 安装

Debian/Ubuntu:

``` shell
apt-get install python-pip
pip install shadowsocks 
```

CentOS:

``` she
yum install python-setuptools && easy_install pip
pip install shadowsocks 
```

有时 Ubuntu 会遇到第一个命令安装 python-pip 时找不到包的情况。pip 官方给出了一个安装脚本，可以自动安装 pip。先下载脚本，然后执行即可：

``` shell
wget https://bootstrap.pypa.io/get-pip.py python get-pip.py 
```
##### 配置文件

创建一个xx.json文件


``` json
{
"server":"your server ip",
"server_port":8388,//客户端连接用的端口号
"local_port":1080,//本机使用的端口号
"password":"your password",
"method":"aes-256-cfb"
"timeout":300
}
```

上面的method是加密方式，选择什么样的加密方式自己确定，或者说看你的客户端支持什么样的假面方式，可选如下：

> aes-128-cfb
> aes-256-cfb
> chacha20
> chacha20-ietf
> aes-128-gcm
> aes-256-gcm
> chacha20-ietf-poly1305

如果想要开启多个端口，把上面的server_port和password改成这样

``` json
"port_password":{//端口:密码 
     "8381":"password1",
     "8382":"password2",
     "8383":"password3",
     "8384":"password4"
    },
```

如果服务器支持ipv6，启用ipv6的话，把上面的server改成`::`

``` json
"server":"::",
```

这时ipv4和ipv6是可以同时使用的。

客户端连接ss的时候，服务器地址填写ipv6的地址，类似于`2402:d0c0:0:133::983d`即可。

##### 启动ssserve

``` shell
ssserver -c xx.json 
```

#### ubuntu内核升级及使用bbr进行网络加速

谷歌新的tcp拥塞控制算法BBR(**Bottleneck Bandwidth and RTT**)



内容出自 https://www.dz9.net/blog/4246.html



众所周知，Ubuntu开启BBR的前提是内核必须等于高于4.9。`uname -a`查看ubuntu内核系统版本，如果满足条件，跳过下一步

##### ubuntu内核升级

`getconf LONG_BIT`查看一下系统是多少位的,

确定系统之后，需要下载必要的升级程序包

http://kernel.ubuntu.com/~kernel-ppa/mainline/

这个网站可以找到最新的程序包，根据自己的需要使用wget命令来下载到服务器；

比如我的服务器是64位，安装4.10.2的内核：

``` shell
wget http://kernel.ubuntu.com/~kernel-ppa/mainline/v4.10.2/linux-image-4.10.2-041002-generic_4.10.2-041002.201703120131_amd64.deb
```

然后切换到你的文件下载目录，执行下列命令来升级：

``` shell
sudo dpkg -i linux-image-4.10.2-041002-generic_4.10.2-041002.201703120131_amd64.deb
```

最后，执行命令`sudo update-grub`，更新grub引导装入程序。

一旦各方面都已完成，重启机器，你就可以准备使用了。系统重启后，打开终端窗口，执行命令uname -a，确保你实际上是在运行你更新之后的内核。

###### 开启TCP BBR

修改系统变量：

```shell
echo "net.core.default_qdisc=fq" >> /etc/sysctl.conf
echo "net.ipv4.tcp_congestion_control=bbr" >> /etc/sysctl.conf
```

保存生效

`sysctl -p`

执行
`sysctl net.ipv4.tcp_available_congestion_control`

如果返回结果
`net.ipv4.tcp_available_congestion_control = bbr cubic reno`
那么恭喜你BBR开启成功了！

也可以执行
`lsmod | grep bbr`
来检测 BBR 是否真的开启成功......

#### v2ray

一键安装脚本

``` shell
apt-get install curl -y bash -c "$(curl -fsSL https://raw.githubusercontent.com/tracyone/v2ray.fun/master/install.sh)"
```

然后看提示就好了。



----

以上