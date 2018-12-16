---
title: CCNA-网络拓扑结构
tags: [CCNA,笔记]
date: 2014-05-19 22:51:00
keywords: [网络拓扑结构]
---

学习过程中的笔记，比较零散

<!--more-->

在设计网络的时候用到qos

在哪些方面评估网络

1 速度 （speed）（路由器借口速度和包转发速度（PPs packet per second））

2开销 （cost）（钱的多少）

3安全 （security）（普通网络路由就行）

4有效性性 （availability）（要求故障时间短（如证券交易））

5可扩展性（scalability）（公司的扩展）

6  可靠性（reliable）

7拓扑结构（topology）（双线备份等）

（总线型  BUS Topology）

（星形 star topology）

（扩展星形）

环形（令牌环网  ring topology）

全网互联（每个交换机相连）（高成本）

部分互联



批量式应用程序（batch application）

类似于文件共享和下载

如FTP  TFTP 

流量大，对延时和丢包不敏感

TCP协议可以重传输





交互式Interactive application

流量小 对延时敏感

如QQ聊天 ssh  mstsc



实时应用程序（real-time application）

对延时和丢包敏感

没有重传输

如 通话和视频（超过150MS  可以感受出来）

UDP协议