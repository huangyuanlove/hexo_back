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

或者看官网 www.v2ray.com   这个需要一些方法才能访问

关键内容如下：

##### 平台支持

V2Ray 在以下平台中可用：

- Windows 7 及之后版本（x86 / amd64）；
- Mac OS X 10.10 Yosemite 及之后版本（amd64）；
- Linux 2.6.23 及之后版本（x86 / amd64 / arm / arm64 / mips64 / mips）；
  - 包括但不限于 Debian 7 / 8、Ubuntu 12.04 / 14.04 及后续版本、CentOS 6 / 7、Arch Linux；
- FreeBSD (x86 / amd64)；
- OpenBSD (x86 / amd64)；
- Dragonfly BSD (amd64)；

##### 下载

预编译的压缩包可以在如下几个站点找到：

1. Github Release: [github.com/v2ray/v2ray-core](https://github.com/v2ray/v2ray-core/releases)
2. Github 分流: [github.com/v2ray/dist](https://github.com/v2ray/dist/)
3. Homebrew: [github.com/v2ray/homebrew-v2ray](https://github.com/v2ray/homebrew-v2ray)
4. Arch Linux: [packages/community/x86_64/v2ray/](https://www.archlinux.org/packages/community/x86_64/v2ray/)
5. Snapcraft: [snapcraft.io/v2ray-core](https://snapcraft.io/v2ray-core)

压缩包均为 zip 格式，找到对应平台的压缩包，下载解压即可使用。

##### 验证安装包

V2Ray 提供两种验证方式：

1. 安装包 zip 文件的 SHA1 / SHA256 摘要，在每个安装包对应的`.dgst`文件中可以找到。
2. 可运行程序（v2ray 或 v2ray.exe）的 gpg 签名，文件位于安装包中的 v2ray.sig 或 v2ray.exe.sig。签名公钥可以[在代码库中](https://raw.githubusercontent.com/v2ray/v2ray-core/master/release/verify/official_release.asc)找到。

#####  Windows 和 Mac OS 安装方式

通过上述方式下载的压缩包，解压之后可看到 v2ray 或 v2ray.exe。直接运行即可。

##### Linux 发行版仓库

部分发行版可能已收录 V2Ray 到其官方维护和支持的软件仓库/软件源中。出于兼容性、适配性考虑，您可以考虑选用由您发行版开发团队维护的软件包或下文的安装脚本亦或基于已发布的二进制文件或源代码安装。
Linux 安装脚本
V2Ray 提供了一个在 Linux 中的自动化安装脚本。这个脚本会自动检测有没有安装过 V2Ray，如果没有，则进行完整的安装和配置；如果之前安装过 V2Ray，则只更新 V2Ray 二进制程序而不更新配置。

以下指令假设已在 su 环境下，如果不是，请先运行 sudo su。

运行下面的指令下载并安装 V2Ray。当 yum 或 apt-get 可用的情况下，此脚本会自动安装 unzip 和 daemon。这两个组件是安装 V2Ray 的必要组件。如果你使用的系统不支持 yum 或 apt-get，请自行安装 unzip 和 daemon


```bash
bash <(curl -L -s https://install.direct/go.sh)
```
此脚本会自动安装以下文件：

* /usr/bin/v2ray/v2ray：V2Ray 程序；
* /usr/bin/v2ray/v2ctl：V2Ray 工具；
* /etc/v2ray/config.json：配置文件；
* /usr/bin/v2ray/geoip.dat：IP 数据文件
* /usr/bin/v2ray/geosite.dat：域名数据文件
此脚本会配置自动运行脚本。自动运行脚本会在系统重启之后，自动运行 V2Ray。目前自动运行脚本只支持带有 Systemd 的系统，以及 Debian / Ubuntu 全系列。

运行脚本位于系统的以下位置：

* /etc/systemd/system/v2ray.service: Systemd

* /etc/init.d/v2ray: SysV

脚本运行完成后，你需要：

1. 编辑 /etc/v2ray/config.json 文件来配置你需要的代理方式；
2. 运行 service v2ray start 来启动 V2Ray 进程；
3. 之后可以使用 service v2ray start|stop|status|reload|restart|force-reload 控制 V2Ray 的运行。

##### go.sh 参数
go.sh 支持如下参数，可在手动安装时根据实际情况调整：

* -p 或 --proxy: 使用代理服务器来下载 V2Ray 的文件，格式与 curl 接受的参数一致，比如"socks5://127.0.0.1:1080" 或 "http://127.0.0.1:3128"。
* -f 或 --force: 强制安装。在默认情况下，如果当前系统中已有最新版本的 V2Ray，go.sh 会在检测之后就退出。如果需要强制重装一遍，则需要指定该参数。
* --version: 指定需要安装的版本，比如 "v1.13"。默认值为最新版本。
* --local: 使用一个本地文件进行安装。如果你已经下载了某个版本的 V2Ray，则可通过这个参数指定一个文件路径来进行安装。
示例：

* 使用地址为 127.0.0.1:1080 的 SOCKS 代理下载并安装最新版本：./go.sh -p socks5://127.0.0.1:1080
* 安装本地的 v1.13 版本：./go.sh --version v1.13 --local /path/to/v2ray.zip
##### Docker
V2Ray 提供了两个预编译的 Docker image：

* v2ray/official https://hub.docker.com/r/v2ray/official/ : 包含最新发布的版本，每周跟随新版本更新；
* v2ray/dev  https://hub.docker.com/r/v2ray/dev/:  包含由最新的代码编译而成的程序文件，随代码库更新；
两个 image 的文件结构相同：
* /etc/v2ray/config.json: 配置文件
* /usr/bin/v2ray/v2ray: V2Ray 主程序
* /usr/bin/v2ray/v2ctl: V2Ray 辅助工具
* /usr/bin/v2ray/geoip.dat: IP 数据文件
* /usr/bin/v2ray/geosite.dat: 域名数据文件



#####  客户端

``` json
{
  "inbounds": [{
    "port": 1080,  // SOCKS 代理端口，在浏览器中需配置代理并指向这个端口
    "listen": "127.0.0.1",
    "protocol": "socks",
    "settings": {
      "udp": true
    }
  }],
  "outbounds": [{
    "protocol": "vmess",
    "settings": {
      "vnext": [{
        "address": "server", // 服务器地址，请修改为你自己的服务器 ip 或域名
        "port": 10086,  // 服务器端口
        "users": [{ "id": "b831381d-6324-4d53-ad4f-8cda48b30811" }]
      }]
    }
  },{
    "protocol": "freedom",
    "tag": "direct",
    "settings": {}
  }],
  "routing": {
    "domainStrategy": "IPOnDemand",
    "rules": [{
      "type": "field",
      "ip": ["geoip:private"],
      "outboundTag": "direct"
    }]
  }
}
```

上述配置唯一要改的地方就是你的服务器 IP，配置中已注明。上述配置会把除了局域网（比如访问路由器）之外的所有流量转发到你的服务器。

##### 服务器配置

``` json
{
  "inbounds": [{
    "port": 10086, // 服务器监听端口，必须和上面的一样
    "protocol": "vmess",
    "settings": {
      "clients": [{ "id": "b831381d-6324-4d53-ad4f-8cda48b30811" }]
    }
  }],
  "outbounds": [{
    "protocol": "freedom",
    "settings": {}
  }]
}
```

服务器的配置中需要确保 `id` 和端口与客户端一致，就可以正常连接了。

#####  运行

- 在 Windows 和 macOS 中，配置文件通常是 V2Ray 同目录下的 `config.json` 文件。直接运行 `v2ray` 或 `v2ray.exe` 即可。
- 在 Linux 中，配置文件通常位于 `/etc/v2ray/config.json` 文件。运行 `v2ray --config=/etc/v2ray/config.json`，或使用 systemd 等工具把 V2Ray 作为服务在后台运行。



----

以上