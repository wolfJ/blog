---
title: 服务集群方案介绍，centos7+lvs+keepalived实践，及原理分析
---
服务(器)集群，涉及的知识点相对偏OS底层些，所以想理解掌握整体的方案，需要把相关的知识点系统的掌握。
本文尽可能的关注解决方案的整体与关键点，弱化介绍方案中各软件应用的参数配置.

##依赖知识点
大致可分为三块：
OS(Centos7，不同系统、版本间会有细微差异)，
网络(协议，拓扑，流向)，
应用(上层应用，如tomcat,mysql,等).

##概念名词
Linux, Cluster, Load Balancer(LB), High Availability(HA), High Performance(HPC), Node, DNS, FailOver, VRRP
Director,RealServer, Proxy
DR,NAT,TUN,
Nginx, Keepalived, LVS, F5, IPVS, KTCPVS, NetFilter
集群，负载，高可用，负载均衡器，节点
扩展性，
二层、三层、四层交换机、路由器，反向代理，中间人
四层交换，七层交换，

##常见问题
方案选型


##场景介绍
![keepalived+nat模式](../images/nat.png)


##场景详解



##参考资料

