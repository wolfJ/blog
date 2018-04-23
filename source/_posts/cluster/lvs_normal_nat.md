---
title: Centos7 LVS+Keepalived 实现负载/高可用之 NAT独立部署及详解  
date: 2018-03-12 11:06  
tags: [linux, centos, lvs, keepalived, vrrp ]  
categories: [技术]  
---


## 目标
LVS + Keepalived 实现负载+高可用，网上也存在许多教程，尝试、排查后大多能跑通。原因是大多教程记录了过程，缺少整体的把握与核心的理解。本篇目的是希望教会五岁的表妹LVS负载。

### 环境
Centos7.3 Minimal版(干净)  
ipvsadm，负载模块  
keepalived，高可用模块  
nginx 应用服务模块  

## 架构
给出架构图:  
![lvs keepalived nat](nat.png)   

<!--more-->
Director节点ipvsadm模块，承担LVS的负载调度能力.  
Director节点keepalived模块，负责VRRP的配置，对故障处理(Failover)实现LVS的高可用.  
RealServer部署自己的服务应用(apache,nginx,tomcat...)，此处选择nginx.   


## 部署

未完待续，敬请期待


## 参考连接
