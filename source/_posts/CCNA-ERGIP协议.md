---
title: CCNA-ERGIP协议
tags: [CCNA,笔记]
date: 2014-05-19 23:07:57
keywords: [ERGIP协议]
---

学习过程中的笔记，比较零散。

<!--more-->

#### EIPGRP（企业级、高级的路由选择协议）

（cisco私有的协议，只有cisco的设备才支持）

特点

* 高级距离矢量型协议（有没有发送路由信息）

* 收敛速度快（DUAL算法，扩散更新算法）

* 支持VLSM和不连续子网

* 部分更新（网络已经收敛，在网络拓扑不出现变动的时候，不在发送更新信息,只向收到变化影响路由条目的路由器发送变化的路由）

* 支持多种网络层协议

* 支持可扩展的网络设计（没有太多的设计要求）

* 使用组播或者单播进行更新（224.0.0.0）

* 可以在网络中的任何一点，进行手工汇总，并且支持超网汇总

* 100%支持无环路的无类路由

* 在wan和lan上容易配置

* 可以支持等开销或者非等开销的负载均衡



#### 工作原理 邻居机制

EIGRP 三张表

邻居(下一跳)表(发送更新之前先搜寻邻居)

邻居表建立之后交换路由信息

拓扑表(从邻居学习来的路由表)(运行DUAL算法,选择最优路径)

路由表（选择出来的最优路径）

##### DUAL 算法

1  选择到达目标网络的最优的无环的路径

2 AD  下一跳路由到达目标网络的开销

3 FD  本地路由到达目标网络的开销=AD+本地路由到达通告给我AD的下一跳路由的开销

lowest－cost＝lowest　FD

（current）successor= 到达目标网络的最优的下一跳路由器

Feasible successor =到达目标网络的次优的下一跳路由器

FC 可行条件  ＡＤ＜ＦＤｍｉｎ

FC 用来防环

EIGRP  数据包直接使用IP协议，协议号88

——hello   发现维护邻接关系（唯一周期性发送的）

当线路>t1  周期5秒  当三个周期以上没有收到hello  认为邻居挂掉

当线路<t1（１。５４４M）   周期30  同上

——update  发送路由更新（路由信息）

——query    向邻居查询特定路由信息

——reply    响应query查询

——ACK     确定数据包

（rtp  确定收到  可靠传输）

![geocoderSearch](/image/ccna/ERGIP_ket_technologies.png)

符合度量值（五种）

带宽  延时 可靠性  负载 MTU

默认使用带宽和延时（可靠性和负载是变值）

K1   k2   k3  k4  k5 

10100(对应五个度量值)

EIGRP建立邻接关系的条件

AS号一样（进程域）

K值

认证

ERGIP　精确通告

Router　　ERGIP　１

no auto-suｍｍary

network　目标IP 反码(与子网掩码相反) (０表示匹配１表示无所谓)

ERGIP的表示是E(跳数最大100 内部管理距离90 外部管理距离170）　

show ipprotocols

show ip eigrp neighbor(邻居表)

show ip eigrp topology

show ip route ergip

show ip ergip traffic

在接口修改

ip hello-interval ergip as号 时间

ip hold-time ergip as号 三倍于hello时间

修改复合度量值　

Metric　weights　０１０１００

第一个只支持０。

检查K值　全网的K值必须一样

![geocoderSearch](/image/ccna/ERGIP_MD5_AUTH.png)

keychain创建好之后每个支持加密的协议都可以用



 EIGRP可以在网络中的任何一点进行路由汇总（支持超网汇总）

在接口级别上进行负载均衡

 bandwidth  。。。。K单位为K  进行负载均衡

EIGRP Load Balancing

默认情况下EIGRP 使用等开销

默认最多在4条等开销路径上进行负载均衡

最多配置到16条路径上进行负载均衡

通过 maximum-paths 命令指定最大的路径

 

非等开销

Unequal-Cost 负载均衡

variance 数字

用法  ：  将最优的FD×variance值，只要大于某个线路的FD   改线路就会自动加进来

![geocoderSearch](/image/ccna/ERGIP_Variance.png)