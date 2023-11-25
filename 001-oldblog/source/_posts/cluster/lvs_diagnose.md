---
title: Linux/Centos7 下排查集群环境问题方法  
date: 2018-03-16 14:21  
tags: [linux, centos7]  
categories: [技术]  
---

在Centos7下部署、配置集群环境时，经常会遇到各种问题，需要一些相关排查技能才可以做到兵来将挡，应对自如。

## 技能介绍

### 如何查看(debug) iptables/netfilter的详细日志

<!--more-->

Load the (IPv4) netfilter log kernel module:

	modprobe nf_log_ipv4
	
Enable logging for the IPv4 (AF Family 2):
	
	sysctl net.netfilter.nf_log.2=nf_log_ipv4
	
reconfigure rsyslogd to log kernel messages (kern.*) to /var/log/kern:

	vi /etc/rsyslog.conf
	#去掉 #kern.* 前面的#号，且指定日志输出的路径。

	> cat /etc/rsyslog.conf | grep -e "^kern"
	kern.*                                /var/log/kern

restart rsyslogd:

	systemctl restart rsyslog

例子	

	#如果想跟踪80端口:
	iptables -t raw -I PREROUTING 1 -p tcp --dport 80 -j TRACE
	iptables -t raw -I OUTPUT 1 -p tcp --sport 80 -j TRACE
	#如果想跟踪ping的协议:
	iptables -t raw -A OUTPUT -p icmp -j TRACE
	iptables -t raw -A PREROUTING -p icmp -j TRACE

检查

	iptables -t raw -L

使用

	tailf /var/log/kern
	Mar 16 02:46:36 localhost kernel: TRACE: mangle:PRE_public_allow:return:1 IN=enp0s3 OUT= MAC=08:00:27:45:5a:f5:08:00:27:56:8a:c4:08:00 SRC=172.16.0.8 DST=172.16.0.11 LEN=52 TOS=0x00 PREC=0x00 TTL=64 ID=35794 DF PROTO=TCP SPT=36842 DPT=80 SEQ=3332505839 ACK=2042156962 WINDOW=229 RES=0x00 ACK RST URGP=0 OPT (0101080A0000000001189425)
	Mar 16 02:46:36 localhost kernel: TRACE: mangle:PRE_public:return:4 IN=enp0s3 OUT= MAC=08:00:27:45:5a:f5:08:00:27:56:8a:c4:08:00 SRC=172.16.0.8 DST=172.16.0.11 LEN=52 TOS=0x00 PREC=0x00 TTL=64 ID=35794 DF PROTO=TCP SPT=36842 DPT=80 SEQ=3332505839 ACK=2042156962 WINDOW=229 RES=0x00 ACK RST URGP=0 OPT (0101080A0000000001189425)
	Mar 16 02:46:36 localhost kernel: TRACE: mangle:PREROUTING:policy:4 IN=enp0s3 OUT= MAC=08:00:27:45:5a:f5:08:00:27:56:8a:c4:08:00 SRC=172.16.0.8 DST=172.16.0.11 LEN=52 TOS=0x00 PREC=0x00 TTL=64 ID=35794 DF PROTO=TCP SPT=36842 DPT=80 SEQ=3332505839 ACK=2042156962 WINDOW=229 RES=0x00 ACK RST URGP=0 OPT (0101080A0000000001189425)
	Mar 16 02:46:36 localhost kernel: TRACE: mangle:INPUT:rule:1 IN=enp0s3 OUT= MAC=08:00:27:45:5a:f5:08:00:27:56:8a:c4:08:00 SRC=172.16.0.8 DST=172.16.0.11 LEN=52 TOS=0x00 PREC=0x00 TTL=64 ID=35794 DF PROTO=TCP SPT=36842 DPT=80 SEQ=3332505839 ACK=2042156962 WINDOW=229 RES=0x00 ACK RST URGP=0 OPT (0101080A0000000001189425)
	

解释说明:

mangle:XXX	//哪张表的哪个规则  
IN=enp0s3 	//接收interface	
OUT=			//出去interface  
MAC=08:00:27:45:5a:f5:08:00:27:56:8a:c4:08:00  //src mac: 08:00:27:56:8a:c4 -> dst mac: 08:00:27:45:5a:f5  
SRC=172.16.0.8 DST=172.16.0.11	//src and dst ip.  


	
	
<!--
#### 问题/目标
#### 架构/思路
#### 解决
#### 分析
-->
