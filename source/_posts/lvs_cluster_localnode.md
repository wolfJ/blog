---
title: LVS：LocalNode / Two Box LVS / LVS集成模式配置与原理详解  
date: 2018-03-04 17:46
tags: [lvs, keepalived, ipvs, vrrp, netfilter, localnode]
categories: [技术]
---

## LVS：LocalNode
LVS经典部署场景是Director服务器与RealServer服务器分开部署，独立服务器，一来是为了充分利用硬件性能，二来是为了避免故障相互影响。  
然而某些时候会把Director与RealServer视为一个软件模块部署在同一台服务器上，用两台服务器部署两套软件，实现负载与高可用。  

### Two Box LVS
仅在两台设备上同样可以完整的实现LVS的故障处理。其中一台设备承担了Director的角色，同时也承担了RealServer的角色。另一台设备是真实服务器。两台设备运行故障处理的代码，允许切换另一台设备为Director角色。两台设备是即实现Director，又有RealServer故障保护功能的最少设备方案了。

```
It's possible to have a fully failover LVS with just two boxes. The machine which is acting as director, also is acting as a realserver using localnode. The second box is a normal realserver. The two boxes run failover code to allow them to swap roles as directors. The two box machine is the minimal setup for an LVS with both director and realserver functions protected by failover.
```

注意：此方案是在内核版本2.6之后，之前也有相应的补丁，具体参见[LVS-HOWTO]()文章。
### 架构图

![two-box-lvs-architecture](two-box-lvs-architecture.png)

#### 集成部署的问题
  
如果集成部署时，仍按照独立方案的配置来，会存在50%的问题  

* client 发请求包给 master director
* 50% 机会 master 把包转给 backup （因为 backup 同时也是 RealServer）
* 因为 backup 的 LVS rules 已经启用，所以50%机会 backup 把包转给 master
* master 收到包后，又可能把包转给 backup，然后陷入死循环。 


### 解决问题实现思路
在两台机器上实现LVS有以下几种思路可以实现：  

* 方案一：对网络包打Mark  
通过iptables对收到的网络包(packets)打Mark。  

* 方案二：更改网络包流向  
通过设置netfilter规则，改变网络包的流向。  

* 方案三：sysctl flag backup_only  
有人提到在 kernel 3.9+ 上，新增了一个 flag: backup_only=1，解决这个问题。但未尝试，不太清楚如果主备切换，flag 怎么做。


### 网络包流向介绍
在此之前先介绍一下网络数据包进入操作系统后的数据流向，了解了内核原理，就能很自然的明白两种思路及其实现了。

Netfilter-packet-flow 数据包流向  
![packet-flow-in-netfilter](Netfilter-packet-flow.jpg)



Netfilter-packet-flow 数据包流向(详细)  
![packet-flow-in-netfilter(详细)](Netfilter-rule-flow.png)

<!-- 111 -->

网络数据包在LVS Director设备中的流向

1. 某块网卡(硬件/Device)接收到数据包(packets)
2. 进入netfilter的INPUT规则链
3. 再进入系统的内核模块IPVS
4. 根据负载技术和调度算法，更改包的目标地址(to RealServer)
5. 出netfilter的OUTPUT规则链


### 一方案：对网络包标记Mark

#### 方案思路  
两台设备相同的架构，当设备接收到请求包，在入口处做标识，区分出是正常请求，还是来自于Director的转发请求。如果是正常请求，则进入转发模块，根据调度策略，转给本设备或另一台设备处理；如果是转发请求，则不需要再进入本设备转发模块了，直接交由RealServer处理。
#### 标记Mark的原理图
![打Mark的原理图](introduce-lvs-mark.png)  

客户端向VIP发出请求或者另设备负载分流请求时，本设备接收到请求，经过防火墙的INPUT过滤模块，mark标记规则：不是来自于另一设备的分流请求就标记mark=3，从另一设备网卡的MAC地址出来的分流请求则不作标记。  
&emsp;&emsp;监控在enp0s3上负载模块，监控条件也改为mark判断，满足mark=3的包，才进入负载模块，不满足的就跳过负载模块。  
&emsp;&emsp;进入负载模块的包，会受调度算法，其它参数配置等影响，可能转向本地的IP地址和端口，从而被本设备的RealServer处理，也可能转向另一设备的IP地址和端口，进而跳出本设备。  
&emsp;&emsp;不进入负载模块的包，因为请求包的目标是VIP，lo上也配置了VIP地址，所以网络包经过了enp0s3后还会被lo的网卡接收到，从而进入用户空间的RealServer模块，处理完后转出本设备。

#### 详细配置

#### 集成部署配置步骤

1. 两台设备上安装基础软件
1. 两台设备上设置Mark规则
1. 两台设备上配置keepalived 
1. 两台设备上配置VIP/VRRP规则

<!--
先后配置主/备、权重、IP负载均衡技术、调度算法、虚拟服务，负载IP等信息、、、、
用于接收目标是VIP的网络包，交由服务应用处理
-->


##### 安装基础软件

Centos7.3-minimal，通过epel源安装基础软件: 

	wget http://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm   
	rpm -ivh epel-release-latest-7.noarch.rpm   
	yum install -y keepalived  
	yum install -y ipvsadm  
	yum install -y nginx  
	
规划IP地址

	#设备A
	RIP: 172.16.0.31  interface: enp0s3
	VIP: 172.16.0.30
	#设备B
	RIP: 172.16.0.32  interface: enp0s3
	VIP: 172.16.0.30	
	
启用服务/开机启动/防火墙设置/SELinux关闭:  

	touch /etc/sysconfig/ipvsadm 
	systemctl enable ipvsadm
	systemctl start ipvsadm
	
	systemctl enable keepalived
	systemctl start keepalived
	
	systemctl enable nginx
	systemctl start nginx
	
	firewall-cmd --add-port=80/tcp --permanent
	firewall-cmd --reload
	firewall-cmd --list-ports
	
	# SELinux disable
	setenforce 0
	getenforce
	# 永久生效: vi /etc/selinux/config
	SELINUX=disabled

区分RealServer的内容

	#设备A 更新nginx首页
	echo 'nginx a' > /usr/share/nginx/html/index.html
	#设备B 更新nginx首页
	echo 'nginx b' > /usr/share/nginx/html/index.html	
	
##### 设置Mark规则
Director A 配置 iptables ，除了 Director B 以外的包，都标记 mark 为3

	# iptables  -t mangle -I PREROUTING -d $VIP -p tcp -m tcp --dport $VPORT -m mac ! --mac-source $MAC_Director_B -j MARK --set-mark 0x3 
	iptables  -t mangle -I PREROUTING -d 172.16.0.30 -p tcp -m tcp --dport 80 -m mac ! --mac-source 08:00:27:E1:3C:C5 -j MARK --set-mark 0x3
	
Director B 配置 iptables ，除了 Director A 以外的包，都标记 mark 为4

	# iptables  -t mangle -I PREROUTING -d $VIP -p tcp -m tcp --dport $VPORT -m mac ! --mac-source $MAC_Director_A -j MARK --set-mark 0x4
	iptables  -t mangle -I PREROUTING -d 172.16.0.30 -p tcp -m tcp --dport 80 -m mac ! --mac-source 08:00:27:77:84:EF -j MARK --set-mark 0x4
 
 查看已经设置的规则
 
 	iptables -vL -t mangle
	


##### 配置keepalived
设备A(Master)配置如下：  
`vi /etc/keepalived/keepalived.conf`

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
            172.16.0.30
        }
    }
    virtual_server fwmark 3 {
        delay_loop 30
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

设备B(Backup)与设备A(Master)基本一致，差异如下：  

	vrrp_instance VI_1 {
		state BACKUP
		priority 50
		...
	}
	
	virtual_server fwmark 4 {
		...
	}

##### 配置VIP/VRRP规则
enp0s3 上的VIP已经由keepalived模块配置好了，不需要我们多做设置，只需要将 lo 上的VIP配置好即可。  
两台设备上都创建此文件 `vi /etc/sysconfig/network-scripts/ifcfg-lo:0`
	
	DEVICE=lo:0
	IPADDR=172.16.0.30
	NETMASK=255.255.255.255
	ONBOOT=yes
	NAME=loopback

修改文件 `vi /etc/sysctl.conf` 添加以下内容 

	# 配有VIP的网卡enp0s3忽略arp包，让lo处理
	net.ipv4.conf.enp0s3.arp_ignore = 1 
	net.ipv4.conf.enp0s3.arp_announce = 2 
	# 允许包转发 
	net.ipv4.ip_forward = 1

使修改生效  

	ifup lo 
	sysctl -p

防火墙设置VRRP接收规则，允许两台设置的keepalived之间通信，实现主/备切换

	# 指定keepalived配置的网卡：enp0s3，固定的VRRP广播地址：224.0.0.18
	firewall-cmd --direct --permanent --add-rule ipv4 filter INPUT 0 --in-interface enp0s3 --destination 224.0.0.18 --protocol vrrp -j ACCEPT
	firewall-cmd --direct --permanent --add-rule ipv4 filter OUTPUT 0 --out-interface enp0s3 --destination 224.0.0.18 --protocol vrrp -j ACCEPT
	firewall-cmd --reload



### 二方案：设置Netfilter规则  

#### 方案思路  
50%的问题是因为网络包进入了两台设备的Director模块。那能不能让网络包不进入两台设备的Director模块，只进某一台设备Director模块呢？Master角色的这台设备肯定会承担所有的网络包入口，Backup角色的这台设备只承担了Master分流过来的网络包请求，Backup的设备在收到网络包，在转入Director模块之前，添加一个规则，让网络包跳过Director模块，直接转到用户空间的RealServer模块处理，这样就不会有50%的问题存在了。

#### 方案原理图

Master角色设备的包流向原理图：
![netfilter原理图](introduce-lvs-netfilter-1.png)   

Backup角色设备的包流向原理图：
![netfilter原理图](introduce-lvs-netfilter-2.png)  

Keepalived的脚本负责维护netfilter中INPUT的规则，根据设备的角色，执行相应的脚本来设置规则或是清理规则。  
当角色为Master时，清除掉规则，所有请求VIP的包，都正常的被LVS的内核模块监听接收处理，按算法或参数，交由本设备的RealServer处理，或是转至另一台设备处理。  
当角色为Backup时，添加规则，所有请求VIP的包，将被强制Redirect至本设备的RealServer处理，处理完后，再转出本设备。  
我们会发现这种方法更加简单，高效些。


#### 集成部署设置步骤

1. 两台设备上安装基础软件  
1. 两台设备上准备脚本  
1. 两台设备上配置keepalived  
1. 两台设备上配置VIP/VRRP规则  


#### 详细配置  

##### 基础安装软件
与方案一相同.  

##### 准备脚本
下载[bypass_ipvs.sh](bypass_ipvs.sh)脚本文件，放于目录`/etc/keepalived/`中。

	# 加执行权限
	chmod +x /etc/keepalived/bypass_ipvs.sh

##### 配置keepalived
设备A (Master)的配置
`vi /etc/keepalived/keepalived.conf`

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
       # Invoked to master transition
       notify_master "/etc/keepalived/bypass_ipvs.sh del 172.16.0.30"
       # Invoked to slave transition
       notify_backup "/etc/keepalived/bypass_ipvs.sh add 172.16.0.30"
       # Invoked to fault transition
       notify_fault "/etc/keepalived/bypass_ipvs.sh add 172.16.0.30"
    }
    virtual_server 172.16.0.30 80 {
        delay_loop 30
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

设备B (Backup)与设备A (Master)基本一致，差异如下：  

	vrrp_instance VI_1 {
		state BACKUP
		priority 50
		...
	}
    


##### 配置VIP / VRRP规则
enp0s3 上的VIP已经由keepalived模块配置好了，不需要我们多做设置。也不需要设置lo的VIP。

修改文件 `vi /etc/sysctl.conf` 添加以下内容 

	# 允许包转发 
	net.ipv4.ip_forward = 1
	
使生效

	sysctl -p

防火墙设置VRRP接收规则，允许两台设置的keepalived之间通信，实现主/备切换

	# 指定keepalived配置的网卡：enp0s3，固定的VRRP广播地址：224.0.0.18
	firewall-cmd --direct --permanent --add-rule ipv4 filter INPUT 0 --in-interface enp0s3 --destination 224.0.0.18 --protocol vrrp -j ACCEPT
	firewall-cmd --direct --permanent --add-rule ipv4 filter OUTPUT 0 --out-interface enp0s3 --destination 224.0.0.18 --protocol vrrp -j ACCEPT
	firewall-cmd --reload



#### 补充说明  
享受成功的喜悦吧：）

多亏了nat规则，我们成功的在两台机器上搭建了负载匀衡和自动故障处理配置。采用这种架构可以完全的最高效的使用可使用的资源。一台设备承担了负载和RealServer角色，当发现这台设备故障时，另一台可以快速切换为负载和RealServer角色。备份设备不仅检查Master设备的状态，同样也处理从负载分发过来的请求。


Keep in mind that the request distribution is not strictly made on a connection basis. The clients connect to one of the realserver going through the VIP. Once this is done and as soon as the realserver in question is available,  further requests will be forwarded to the same realserver. In a classical scenario, many clients connect to the VIP, thus the global amount of requests is equally distributed between the two nodes.


### 调试排错
如有需要，随后附上调试排错的方法，和详细步骤。  
例如：  
在测试过程中，客户端一直没有响应？  
或者为什么请求一直不进入负载模块，等。。。？  

### 参考
[Building Two-Node Directors/Real Servers using LVS and Keepalived](http://kb.linuxvirtualserver.org/wiki/Building_Two-Node_Directors/Real_Servers_using_LVS_and_Keepalived)  

[LVS DR模式的一些问题](http://linbo.github.io/2017/08/20/lvs-dr)  

[Failover and loadbalancer using keepalived (LVS) on two machines -ok](http://gcharriere.com/blog/?p=339)  

[Linux内核参数之arp_ignore和arp_announce](https://www.jianshu.com/p/734640384fda)
