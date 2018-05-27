---
title: SSH免密登录-最佳实践
date: 2018-05-27 20:11
tags: [ssh, linux, mac, shell ]
categories: [技术]
---

# SSH免密登录-最佳实践

有许多场景需要使用Terminal(SSH)登录远程主机，每次都需要输入密码，这已经严重影响到自动化流程与交互体验了，不能接受！  
* Mac OS下terminal, iTerm2等客户端SSH远程主机场景
* Linux (Ubuntu, CentOS ...)下Terminal客户端SSH远程主机场景

### 已有方案

* 基于SSH keys 公私钥机制：[SSH keys 介绍](https://wiki.archlinux.org/index.php/SSH_keys_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87))
* 基于Expect脚本：[Expect 介绍](https://www.cnblogs.com/lzrabbit/p/4298794.html)
* 使用sshpass工具: [sshpass 介绍](https://blog.csdn.net/Alex_luodazhi/article/details/50177707)
* Mac下iTerm2的密码Trigger机制

<!--more-->

网上对这几种方案都有详细介绍，这里只是大概的分析比较：

* SSH keys，安全性高，操作步骤相对繁琐一些  
* Expect，是处理交互的命令，写的SSH登录脚本，难懂，不便与其它代码集成，耦合度高，存在明文密码风险  
* sshpass，需安装，用法、流程非常简单，也存在明文密码风险  
* iTerm2的Trigger机制，先维护密码，不彻底、不好用，不推荐

### 推荐方案
考虑到安全性、易用性、可维护性等，在SSH keys方案之上，提供自动脚本处理机制，解决操作步骤繁琐的问题。

只需要在主机上操作执行即可：  
创建ssh-key的自动脚本 `vi ~/ssh-key.sh`

    #!/bin/bash    
    function initKeyAndCopyToDest() {
      if [ ! -e ~/.ssh/id_rsa.pub ]; then
        ssh-keygen -P ''
      fi
    
      HOST_IP=$1
      cat ~/.ssh/id_rsa.pub|ssh root@${HOST_IP} "mkdir -p /root/.ssh/; cat >> /root/.ssh/authorized_keys"
    }
    
    if [ $1 ]; then
    	initKeyAndCopyToDest $1
    else
    	echo -e "Usage: `basename $0` ip"
    fi
    
添加执行权限: `sudo chmod +x ~/ssh-key.sh`   

第一次连接前，先执行下脚本，第一次执行脚本时，会检查之前有没有生成过ssh-key，如果没有，直接输入回车完成创建，然后会提示输入target-ip的密码：

    ~/ssh-key.sh target-ip
    
成功后可通过 `ssh root@target-ip date` 检查配置成功否。

同样，scp 远程copy文件命令也可以不用密码了，就只可以方便的编写，从一台机器将内容同步至多台机器的功能了。

### ssh-key脚本分析
1. 检查~/.ssh/id_rsa.pub文件是否存在，是否已经创建过ssh密钥文件，
2. 如果未创建，则创建，直接输入回车即可
3. 读取公钥文件，通过管道，作下一步的参数
4. ssh连接目标机器，提示输入密码
5. 创建目录
6. 将本机的id_rsa.pub内容，追加到目标主机authorized_keys中

