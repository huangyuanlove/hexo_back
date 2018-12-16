---
title: CCNA-ospf协议
tags: [CCNA,笔记]
date: 2014-05-19 22:59:57
keywords: [OSPF]
---

学习过程中的笔记，比较零散

<!--more-->

开放式最短路径优先  open 最短路径优先算法 

高级企业级路由协议

链路状态型的路由选择协议

每个路由的选择是每个路由器独立选择的   不是靠别的路由器告诉的

DU（距离矢量型） 交换的是route

LS （链路状态型）交换的是linkstate（描绘网络拓扑的信息）

链路状态型

拥有全网的路由信息,并且每个路由器的路由信息都是一致的  

链路状态信息通过LSA数据结构进行描述 

路由器之间相互交换链路好状态信息,构成LSDB(拓扑数据库 链路状态数据库)

运行spf  构建spftree  触发更新  部分有界更新

OSPF 维护三张表

1邻居表

2拓扑表

3路由表   

建立邻居关系，相互交换LSDB建立拓扑表（网络地图），基于拓扑表运行SPF算法，选择最优路径，构建路由表。

![链路状态型](/image/ccna/link-state_routing_protocels.png)

运行两次SPF算法，第一次构建全局拓扑表，去掉环路，第二次运行时删除次优路径，构建路由表。

OSPF算法需要rourte-id唯一标识每一台路由器

 

使用环回接口做为route-id的原因是：

环回接口并不是实际通信接口，指定起来比较方便。一台路由器上至少要有一个IP才能开启OSPF算法

![链路状态型](/image/ccna/link-state_routing_protocels_router_id.png.png)

