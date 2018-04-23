---
title: Centos7 LVS+Keepalived 实现负载/高可用之 DR 独立部署及详解  
date: 2018-03-8 15:06  
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
![lvs keepalived dr](dr.png)   

<!--more-->
Director节点ipvsadm模块，承担LVS的负载调度能力.  
Director节点keepalived模块，负责VRRP的配置，对故障处理(Failover)实现LVS的高可用.  
RealServer部署自己的服务应用(apache,nginx,tomcat...)，此处选择nginx.   

## 部署

### 设备安装基础软件

Centos7 采用epel源安装相关软件及依赖

	wget http://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm   
	rpm -ivh epel-release-latest-7.noarch.rpm   


两台Director设备(主/备)，安装ipvsadm，keepalived 开机启动/启动服务
	
	yum install -y keepalived  
	yum install -y ipvsadm  

	#启动时需要一个空文件，否则异常
	touch /etc/sysconfig/ipvsadm 
	systemctl enable ipvsadm
	systemctl start ipvsadm
	
	systemctl enable keepalived
	systemctl start keepalived


两台Director设备 SELinux关闭(否则keepalived启动异常):  
		
	# SELinux disable
	setenforce 0
	getenforce
	# 永久生效: vi /etc/selinux/config
	SELINUX=disabled

两台RealServer设备 安装nginx
	
	yum install -y nginx  

	systemctl enable nginx
	systemctl start nginx
	
	#设备A 更新nginx首页，区分服务
	echo 'nginx a' > /usr/share/nginx/html/index.html
	#设备B 更新nginx首页，区分服务
	echo 'nginx b' > /usr/share/nginx/html/index.html	

每台设备添加防火墙端口(如果启动用firewalld，也可关闭防火墙服务)

	firewall-cmd --add-port=80/tcp --permanent
	firewall-cmd --reload
	firewall-cmd --list-ports


### Director设备上配置keepalived

在Director设备上配置keepalived，先后配置主/备、权重、IP负载均衡技术、调度算法、监听IP，负载IP等信息。

Director设备 Master 配置，编辑配置文件 `vi /etc/keepalived/keepalived.conf`
	
	global_defs {
	   router_id LVS_DEVEL
	}
	vrrp_instance VI_1 {
	    state MASTER
	    interface enp0s3
	    virtual_router_id 51
	    priority 100
	    advert_int 1
	       authentication {
	        auth_type PASS
	        auth_pass 1111
	    }
	    virtual_ipaddress {
	        172.16.0.30/24
	    }
	}
	virtual_server 172.16.0.30 80 {
	    delay_loop 6
	    lb_algo rr
	    lb_kind DR
	    persistence_timeout 0
	    protocol TCP
	
	    real_server 172.16.0.31 80 {
	        TCP_CHECK {
	            connect_timeout 3
	        }
	    }
	    real_server 172.16.0.32 80 {
	        TCP_CHECK {
	            connect_timeout 3
	        }
	    }
	}
	
Director设备 Backup 与设备 Master 基本一致，差异如下：

	vrrp_instance VI_1 {
	    state BACKUP
	    priority 50
	    ...
	}

### 设备上配置VIP/VRRP规则  
Director设备的VIP已经由Keepalived配置文件中配置好了，重新启动下keepalived模块即可: `systemctl restart keepalived `。   
Director设备仍需要设置防火墙VRRP接收规则，允许两台设置的keepalived之间通信，实现主/备切换

	# 指定keepalived配置的网卡：enp0s3，固定的VRRP广播地址：224.0.0.18
	firewall-cmd --direct --permanent --add-rule ipv4 filter INPUT 0 --in-interface enp0s3 --destination 224.0.0.18 --protocol vrrp -j ACCEPT
	firewall-cmd --direct --permanent --add-rule ipv4 filter OUTPUT 0 --out-interface enp0s3 --destination 224.0.0.18 --protocol vrrp -j ACCEPT
	firewall-cmd --reload


RealServer配置VIP，用于接收目标是VIP的网络包

新建一个脚本 lvs-web.sh: `vi /etc/init.d/lvs-web.sh`  
	
	#!/bin/sh
	VIP=172.16.0.30
	. /etc/rc.d/init.d/functions
	
	case "$1" in
	 
	start)
	    /sbin/ifconfig lo down
	    /sbin/ifconfig lo up
	    echo "1" >/proc/sys/net/ipv4/conf/lo/arp_ignore
	    echo "2" >/proc/sys/net/ipv4/conf/lo/arp_announce
	    echo "1" >/proc/sys/net/ipv4/conf/all/arp_ignore
	    echo "2" >/proc/sys/net/ipv4/conf/all/arp_announce
	    /sbin/sysctl -p >/dev/null 2>&1
	    /sbin/ifconfig lo:0 $VIP netmask 255.255.255.255 up   
	    /sbin/route add -host $VIP dev lo:0
	    echo "LVS-DR real server starts successfully.\n"
	    ;;
	stop)
	    /sbin/ifconfig lo:0 down
	    /sbin/route del $VIP >/dev/null 2>&1
	    echo "1" >/proc/sys/net/ipv4/conf/lo/arp_ignore
	    echo "2" >/proc/sys/net/ipv4/conf/lo/arp_announce
	    echo "1" >/proc/sys/net/ipv4/conf/all/arp_ignore
	    echo "2" >/proc/sys/net/ipv4/conf/all/arp_announce
		echo "LVS-DR real server stopped.\n"
	    ;;
	status)
	    isLoOn=`/sbin/ifconfig lo:0 | grep "$VIP"`
	    isRoOn=`/bin/netstat -rn | grep "$VIP"`
	    if [ "$isLoON" == "" -a "$isRoOn" == "" ]; then
	        echo "LVS-DR real server has run yet."
	    else
	        echo "LVS-DR real server is running."
	    fi
	    exit 3
	    ;;
	*)
	    echo "Usage: $0 {start|stop|status}"
	    exit 1
	esac
	exit 0

依赖脚本权限：`chmod +x /etc/rc.d/init.d/functions`  
使用：`sh /etc/init.d/lvs-web.sh start|stop|status`



至此，完成所有配置，享受短暂的成功喜悦吧：）

### 如何排查问题
点击[Linux下如何排查问题]()

## 分析
DR模式下，请求的包流向是怎么走的？  
在Director设备上请求VIP会怎么样？  
在RealServer设备上请求VIP会怎么样？  

#### 在Director设备上请求VIP会怎么样？
独立部署时，在director设备上作为Client发起请求是怪异的做法，最多是在RealServer上作为Client发起请求，如果存在，发现不能正常的收到响应的包。

## 补充知识点

* How LVS-DR works
* LVS clients on Realservers?
* Handling the arp problem for LVS-DR
* 6.8.2.masquerade the service(s) for the realservers
* 6.8.3. add a default route for packets from the primary IP on the outside of the director


* rewriting, re-mapping, translating ports with iptables in LVS-DR

With LVS-NAT you can rewrite ports: `/sbin/ipvsadm -a -t VIP:http -r RIP:9999 -m -w 1`  [see Re-mapping ports with LVS-NAT](http://www.austintek.com/LVS/LVS-HOWTO/HOWTO/LVS-HOWTO.LVS-NAT.html#re-mapping_ports_lvs_nat)

However, LVS-DR and LVS-Tun just forward the packets to the realserver without rewriting the ports. You can still rewrite the ports at the realserver with **iptables** allowing the realserver to listen on another port.


	/sbin/iptables -t nat -A PREROUTING -d VIP -p tcp -m tcp --dport 80 -j DNAT --to-destination VIP:9999


#### LVS clients on Realservers



## 参考连接

[LVS/DR模式原理剖析](http://zh.linuxvirtualserver.org/node/2585)
