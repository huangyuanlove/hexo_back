---
title: CCNA-DHCP和NAT
tags: [CCNA,笔记]
date: 2014-05-19 23:07:57
keywords: [DHCP,NAT]
---

学习过程中的笔记，比较零散

<!--more-->

#### 动态主机配置协议：dynamic host configuration protocol  架构为C/S 客户端/服务器   前身是 boot protocol

DHCP使用UDP协议，端口号为67(客户端)，68（服务端）

IP可以和mac绑定

工作过程如下：

C ---->Ｓ（discover message）发现消息，在局域网内广播发送

S ---->Ｃ（offer message） 回复消息  单播响应请求的客户端

C ---->Ｓ（request message）请求消息  广播发送（因为一个局域网可能有多个路由器，先收到哪个服务器的offer就用谁的IP）

S ---->Ｃ (acknowledge message) 确认消息  ACK消息

DHCP　configuration　Ｏｎ　IOS

（dhcp只能配置在以太网上，串口是没有办法配置dhcp的）



| （config）#service dhcp                         | 启动DHCP服务       |
| ----------------------------------------------- | ------------------ |
| (config)# ip dhcp pool name                     | 指定地址池的名字   |
| (dhcp-config)#network 192.168.1.0 255.255.255.0 | 指定为哪个网段服务 |
| (dhcp-config)#default-router 192.168.1.254      | 可选，指定网关     |
| (dhcp-config)#domain-name name      | 可选，指定域名     |
|(dhcp-config)#dns-server IP IP|可选，DNS服务器|
|(dhcp-config)#lease 7|可选，租约期，单位是天|
|config）# ip dhcp excluded-address 192.168.1.1 192.168.1.10|排除不想分配的IP地址|
#### NAT： 网络地址转换，用来修改IP数据包中的源、目的地址

##### 私有IP地址

A类  10.0.0.0 -- 10.255.255.255

B类 172.16.0.0 -- 172.31.255.255

C类 192.168.0.0 -- 192.168.255.255

只能存在于内网中，是不能出现在互联网上的

##### 为何使用NAT技术

1 节省IP地址 （NAT + VLSM/CIDR）

2 安全考虑，隐藏内部真实的IP地址

3 NAT TCP负载均衡

4 解决地址冲突问题（公司合并）

##### 缺点：

1 影响路由器的转发性能（额外修改IP地址，计算校验和等）

2 破坏了IP的端到端特性

3 与很多安全相关协议不兼容（IPSec等）

##### NAT  分类

1 静态NAT

--手工配置NAT映射表

-- 一对一转换

2 动态NAT

-- 定义地址池，动态创建NAT映射表

-- 一对一转换

3 PAT（NAT Overload NAT过载） 就是我们平时说的NAT转换

-- 多对一转换

-- 通过端口号标识不同数据流（和通信的端口号没有关系）

![nat网络拓扑](/image/ccna/nat.png)

模拟网络连接如图：

C1、C2和NAT服务器在192.168.1.0/24的网段，其中NAT服务器的IP是192.168.1.254，NAT服务器通过窗口和ISP相连，公网地址在61.1.1.0/24网段

配置名命令（没有全拼，自己补全即可）

（需要指定转换的方向，即哪边是内网，哪边是外网）

① 静态NAT ，一对一关系(不要忘了指定转换方向，即哪边是内网，哪边是外网)

在 c1上的配置

1）指定接口的IP的地址，并指定缺省网关

conf t

int f0/0

ip add 192.168.1.1 255.255.255.

no sh

ip route 0.0.0.0 0.0.0.0 192.168.1.254

在C2上

指定接口的IP，并指定缺省网关

conf t

int f0/0

ip add 192.168.1.2 255.255.255.

no sh

ip route 0.0.0.0 0.0.0.0 192.168.1.254

在NAT服务器上

制定IP地址，并进行配置，命令如下

Conf t

Int f0/0

Ip add 192.168.1.254 255.255.255.0

No sh

Int s1/1  (在和ISP相连的接口上配置IP)

Ip add 61.1.1.1 255.255.255.0

Cl ra 64000

No sh 

End

(NAT映射)

Ip nat inside source static 192.168.1.1(源IP地址) 61.1.1.11(目标IP地址)

Ip nat inside source static 192.168.1.2(源IP地址) 61.1.1.12(目标IP地址)

(可以通过 show ip nat traslations查看转换表)

Int fa0/0

Ip nat inside(指定为内网，即入口)

Int s1/1

Ip nat outside（指定为外网，即出口）

3 在ISP上

指定和NAT相连的接口Ip

Conf t 

Int s1/1

Ip add 61.1.1.2 255.255.255.0

Cl ra 64000 

No sh 

End

 

②动态NAT(不要忘了指定转换方向，即哪边是内网，哪边是外网)

在nat上

Ip nat pool pool-name 61.1.1.10 61.1.1.20 netmask 255.255.255.0

(定义访问控制列表)

Access-list 1 permit 192.168.1.0 0.0.0.255

Ip nat inside source list 1 pool pool-name

在C1、c2和IPS上的配置和静态nat相同

 

 

③NAT过载 （多个内网对应一个外网）(不要忘了指定转换方向，即哪边是内网，哪边是外网)

Access-list 1 permit 192.168.1.0 0.0.0.255

Ip nat inside source list 1 pool pool-name

 

Ip nat inside source list 1 interface s1/1 overload