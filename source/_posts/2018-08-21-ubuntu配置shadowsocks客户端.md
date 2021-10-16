---
title: Ubuntu 配置 shadowsocks 客户端
date: 2018-08-21 20:24:00
categories:
 - "VPN"
tags:
 - "ubuntu"
 - "shadowsocks"
 - "privoxy"
---
# 前言

在日常的工作学习中，经常需要搭建各种环境，而很多环境都是由国外开发并开源的，有一些软件或源码必须要到墙外面才能够下载，所以需要在自己的环境中配置 shadowsocks 客户端来连接国外的网络。在这里将介绍如何在 Ubuntu 系统下搭建 shadowsocks 客户端。

<!--more-->
------

# 安装 shadowsocks

1. 安装 python pip工具

   ```
   $ sudo apt install python-pip
   ```

1. 安装 shadowsocks

   ```
   $ sudo pip install shadowsocks
   ```

2. 配置 shadowsocks

   ```
   $ sudo vi /etc/shadowsocks.json
   ```

   输入以下 json 格式的代码：

   ```
   {
       "server": "服务器ip",
       "server_port": 服务器端口,
       "password": "你的密码",
       "method": "aes-256-cfb",
       "timeout": 300
   }
   ```

3. 启动 shadowsocks 服务

   ```
   $ sslocal -c /etc/shadowsocks.json &
   ```

   > 加上 & 以让 shadowsocks 进程在后台运行

4. 设置 shadowsocks 开机自启动

   将启动服务的命令添加到 `/etc/rc.local` 文件中的 `exit 0` 之前

   ```
   $ sudo vi /etc/rc.local
   ……
   sslocal -c /etc/shadowsocks.json &
   exit 0
   ```

以上就是SS的搭建了，这个时候我们发现上网时并不可以翻墙，原因是需要将sock5代理映射为http代理。代理的软件很多，我选择了推荐度比较高的privoxy，下面是privoxy的配置。

------

# 安装 privoxy

1. 安装 privoxy

   ```
   $ sudo apt install privoxy
   ```

1. 配置 privoxy

   打开 `/etc/privoxy/config`

   ```
   $ sudo vi /etc/privoxy/config
   ```

   找到其中的4.1节，看一下有没有一句`listen-address localhost:8118`的代码，如果被注释了，取消注释。因为版本不一样这句的状态可能会不一样。 然后再将 `localhost` 改成 `127.0.0.1`（这一步很重要，反正我因为这一步的设置问题搞了很久都不知道为什么连不上外网），如图所示：

   ![privoxy配置图](https://raw.githubusercontent.com/cao0507/My-Pictures-Repository/master/blog/VPN%E9%85%8D%E7%BD%AEprivoxy%20%E5%9B%BE%E4%B8%80.png)

   接着找到5.2节，在本节末尾加入代码 `forward-socks5t / 127.0.0.1:1080 .`，注意最后有一个点号，如图：

   ![privoxy配置图](https://raw.githubusercontent.com/cao0507/My-Pictures-Repository/master/blog/VPN%E9%85%8D%E7%BD%AEprivoxy%20%E5%9B%BE%E4%BA%8C.png)

2. 重启 privoxy 服务

   ```
   $ sudo /etc/init.d/privoxy restart
   ```

3. 设置开机自启动 privoxy 服务

   将启动服务的命令添加到 `/etc/rc.local` 文件中的 `exit 0` 之前：

   ```
   $ sudo vi /etc/rc.local
   ……
   sslocal -c /etc/shadowsocks.json &
   /etc/init.d/privoxy start
   exit 0
   ```

4. 代理配置

   ```
   $ sudo vi /etc/profile
   export http_proxy=http://127.0.0.1:8118
   export https_proxy=http://127.0.0.1:8118

   $ source /etc/profile
   ```

------

# 测试

```
curl www.google.com
```

或

```
wget www.google.com
```


