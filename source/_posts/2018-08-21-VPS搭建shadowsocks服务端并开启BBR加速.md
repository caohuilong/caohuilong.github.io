---
title: Vultr CentOS搭建shadowsocks服务端并开启BBR加速
date: 2018-08-21 19:34:00
categories:
 - "VPN"
tags:
 - "centos"
 - "shadowsocks"
 - "BBR"
---
# 前言

最近教研室很多同学来问有没有公用的 VPN，教研室以前有师兄去买过一些 VPN，但现在师兄也毕业了就用不了了。为了同学们的方便，作为网管应该尽力满足大家日常查阅资料的需求，于是向老师申请了经费去购买了一台 VPS 来搭建教研室公用的 VPN。
<!--more-->

# 购买 VPS

我这里购买的是 vultr 的 VPS，在新加坡的节点，每个月5美元，其实是按小时计费，每小时0.007美元，如果出问题了可以方便的停止购买，不像其他厂商按年一次性付费的话，万一不能用了就很亏。操作系统选择 CentOS 7 64位。

# 安装过程

### 安装 Shadowsocks 服务

shadowsocks有很多版本，如Python，node.js，libev，Go等，Python版本用的人是最多的，但很久没有更新了，这里选择 Go 版本的shadowsocks。

在安装shadowsocks之前需要先**安装 Go 语言的环境**：

- 从官网下载 Linux 平台的源码包

  ```
  # wget https://dl.google.com/go/go1.10.3.linux-amd64.tar.gz
  ```

- 解压到指定目录

  ```
  # sudo tar zxvf go1.10.3.linux-amd64.tar.gz -C /usr/local/
  ```

- 配置环境变量

  ```
  # vi .bashrc
  添加
  export PATH=$PATH:/usr/local/go/bin
  ```

  保存并使生效：

  ```
  # source .bashrc
  ```

**安装 shadowsocks**，使用一键安装脚本（https://github.com/iMeiji/shadowsocks_install）

```
# wget --no-check-certificate https://raw.githubusercontent.com/iMeiji/shadowsocks_install/master/shadowsocks-go.sh
# chmod +x shadowsocks-go.sh
# ./shadowsocks-go.sh 2>&1 | tee shadowsocks-go.log
```

**卸载 shadowsocks 方法：**

```
# ./shadowsocks-go.sh uninstall
```

**shadowsocks 常用命令：**

- 启动：`/etc/init.d/shadowsocks start `
- 停止：`/etc/init.d/shadowsocks stop`
- 重启：`/etc/init.d/shadowsocks restart` 
- 状态：`/etc/init.d/shadowsocks status`

执行完前面的安装脚本后，查看 shadowsocks 的运行状态：

```
# /etc/init.d/shadowsocks status
shadowsocks-go running with PID 1629
```

能看到进程 ID 说明 shadowsocks 服务已经在运行了。

**配置 shadowsocks 开机自启动：**

```
# vi /etc/rc.local
添加
/etc/init.d/shadowsocks restart
```

这样在系统重启后就可以自动加载 shadowsocks 服务了。

**配置防火墙：**

检查防火墙是否允许你设定的端口进行通信

```
# iptables -vnL | grep 8989
```

如果没有信息的话，就是防火墙不允许该端口进行通信。 

需设置：

```
# firewall-cmd --zone=public --add-port=8989/tcp --permanent
# firewall-cmd --reload
```

> 由于 CentOS 7 默认安装的是 firewalld，并没有安装 iptables-services，不能使用 iptables-save 来保存iptables 规则并在下次启动时自动加载，所以上面使用是 firewalld 来配置永久规则，这样在关机重启后规则也不会消失。

到这里，shadowsocks 服务器就基本上已经配置好了，你可以使用客户端来上外网了。但这时候的网络连接的速度可能只能够保证查查网页，如果要下载或者看 YouTube 速度很慢，所以后将进行配置TCP 加速。

---

### TCP 加速

在后面会升级系统内核，所以先查看一下服务器的内核版本：

```
# uname -a
Linux vultr.guest 3.10.0-862.3.2.el7.x86_64 #1 SMP Mon May 21 23:36:36 UTC 2018 x86_64 x86_64 x86_64 GNU/Linux
```

加速有锐速加速和 Google BBR 加速，这里使用 BBR 加速。TCP BBR是谷歌出品的 TCP 拥塞控制算法，目的是要尽量跑满带宽，并且尽量不要有排队的情况。BBR 可以起到单边加速 TCP 连接的效果。

Google提交到Linux主线并发表在ACM  queue期刊上的TCP-BBR拥塞控制算法。继承了Google“先在生产环境上部署，再开源和发论文”的研究传统。TCP-BBR已经再YouTube服务器和Google跨数据中心的内部广域网(B4)上部署。由此可见出该算法的前途。 

TCP-BBR的目标就是最大化利用网络上瓶颈链路的带宽。一条网络链路就像一条水管，要想最大化利用这条水管，最好的办法就是给这跟水管灌满水。 

BBR解决了两个问题： 

- 在有一定丢包率的网络链路上充分利用带宽。非常适合高延迟，高带宽的网络链路。 
- 降低网络链路上的buffer占用率，从而降低延迟。非常适合慢速接入网络的用户。 

Google 在 2016年9月份开源了他们的优化网络拥堵算法BBR，最新版本的 Linux内核(4.9-rc8)中已经集成了该算法。 

**一键安装脚本：**

```
# wget --no-check-certificate https://github.com/teddysun/across/raw/master/bbr.sh
# chmod +x bbr.sh
# ./bbr.sh
```

安装完成后会提示重启，重启完成后，查看内核：

```
# uname -r
4.18.3-1.el7.elrepo.x86_64
```

高于 4.9 就可以了

检查是否开启 BBR：

```
# sysctl net.ipv4.tcp_available_congestion_control
net.ipv4.tcp_available_congestion_control = reno cubic bbr

# sysctl net.ipv4.tcp_congestion_control
net.ipv4.tcp_congestion_control = bbr

# sysctl net.core.default_qdisc
net.core.default_qdisc = fq

# lsmod | grep bbr
tcp_bbr                20480  7
#返回值有 tcp_bbr 则说明已经启动
```

完成以上步骤，则 TCP 加速也已经配置好了，接下来就可以体验飞快的下载速度以及 1080p 的高清视屏啦！

shadowsocks 客户端下载连接：[Shadowsocks - Clients](https://shadowsocks.org/en/download/clients.html) 
