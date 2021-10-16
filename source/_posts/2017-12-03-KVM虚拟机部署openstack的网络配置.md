---
title: KVM虚拟机部署openstack的网络配置
date: 2017-12-03 12:34:35
categories: 
  - "KVM"
tags:
  - "KVM"
  - "Openstack"
  - "vnc"
  - "ubuntu"
  - "Virtual Machine"
---
# 概述

这篇文章记录的是按照官方文档在**KVM环境**下部署**双节点openstack**过程中，前期准备KVM环境和网络配置相关的内容，在完成这篇博客涉及到的工作之后就可以按照官方文档手动安装openstack了。本文涉及的主要工作，首先是在服务器的**ubuntu 16.04 desktop版**系统上搭建kvm环境，然后在服务器上安装VNC远程桌面，最后在KVM环境中开启两台虚拟机，分别两张网卡，第一张网卡使用**桥接模式**，第二张网卡使用**NAT模式**。下面开始介绍一下这个过程。

<!--more-->

# 服务器搭建KVM环境

## 查看CPU是否支持KVM

`$ egrep -c "(svm|vmx)" /proc/cpuinfo`

输出结果大于0证明CPU支持KVM虚拟化 

![虚拟CPU个数](https://raw.githubusercontent.com/cao0507/Pictures/master/blog/infocpu.png)

## 安装KVM及相关依赖包

`$ sudo apt-get install qemu-kvm qemu virt-manager virt-viewer libvirt-bin bridge-utils`

## 启用桥接网络

在服务器上启用桥接网络需要配置一个桥接设备br0，配置br0有两种方式，通过手动配置和通过修改文件配置。 

### 通过手动配置

- 创建br0网桥

   `# brctl addbr br0`

- 将eth0网卡添加到br0上，此时可能会断网

  `# brctl addif br0 eth0`

- 删除eth0上的IP地址

  `# ip addr del dev eth0 192.168.1.25/24 `

- 配置br0的IP地址并启动br0网桥设备

  `# ifconfig br0 192.168.1.25/24 up`

- 重新加入默认网关

  `# route add default gw 192.168.1.1`

- 查看配置是否生效 

  ```
  # route     //查看默认网关，输出结果如下
  Kernel IP routing table
  Destination             Gateway             Genmask         Flags     Metric      Ref    Use     Iface
  default              192.168.1.1             0.0.0.0          UG        0          0      0        br0
  192.168.1.0             *                 255.255.255.0       U         0          0      0        br0
  ```

  ```
  # ifconfig     //查看eth0和br0的IP信息，输出结果如下，可以发现现在br0有IP而eth0没有IP了
  br0       Link encap:Ethernet  HWaddr 00:e0:81:e2:3c:3d  
            inet addr:192.168.1.25  Bcast:192.168.1.255  Mask:255.255.255.0
            inet6 addr: fe80::2e0:81ff:fee2:3c3d/64 Scope:Link
            UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
            RX packets:1316822 errors:0 dropped:5787 overruns:0 frame:0
            TX packets:365475 errors:0 dropped:0 overruns:0 carrier:0
            collisions:0 txqueuelen:1000
            RX bytes:581279124 (581.2 MB)  TX bytes:562586852 (562.5 MB)
   
  eth0      Link encap:Ethernet  HWaddr 00:e0:81:e2:3c:3d  
            inet6 addr: fe80::2e0:81ff:fee2:3c3d/64 Scope:Link
            UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
            RX packets:6671034 errors:0 dropped:9627 overruns:0 frame:0
            TX packets:840972 errors:0 dropped:0 overruns:0 carrier:0
            collisions:0 txqueuelen:1000
            RX bytes:1346523816 (1.3 GB)  TX bytes:614510541 (614.5 MB)
            Memory:dfb80000-dfbfffff
   
  lo        Link encap:Local Loopback  
            inet addr:127.0.0.1  Mask:255.0.0.0
            inet6 addr: ::1/128 Scope:Host
            UP LOOPBACK RUNNING  MTU:65536  Metric:1
            RX packets:1450290 errors:0 dropped:0 overruns:0 frame:0
            TX packets:1450290 errors:0 dropped:0 overruns:0 carrier:0
            collisions:0 txqueuelen:1000
            RX bytes:24027042487 (24.0 GB)  TX bytes:24027042487 (24.0 GB)
  ```

这就是通过手动来配置桥接设备br0的方法，这种方法在配置好之后马上就生效了，但是在系统重启之后这些配置信息都会被清除，要想使配置永久生效则需要修改网络配置文件，也就是下面的方法。 

### 通过修改文件配置

- 修改前先将网络配置文件进行备份

  `# cp  /etc/network/interfaces  /etc/network/interfaces.bak`

- 修改网络配置文件`/etc/network/interfaces`

  `# vi  /etc/network/interfaces    //修改结果如下`

  ```
  auto lo
  iface lo inet loopback
   
  # Enabing Bridge networking br0 interface
  auto br0
  iface br0 inet static
  address 192.168.1.25
  network 192.168.1.0
  netmask 255.255.255.0
  broadcast 192.168.1.255
  gateway 192.168.1.1
  dns-nameservers 223.5.5.5
  bridge_ports eth0
  bridge_stp off
  ```

保存后退出，关机重启中配置文件就生效了。这种方法只需要修改配置文件然后重启就可以，比较简单，而且是永久生效，比较符合我们的需求，因为我们的虚拟机通过桥接模式连接外网的话都是连接到br0上的。 

## 修改virbr0的网段

在服务器上安装好虚拟化软件后，KVM会自动生成一个`virbr0`的桥接设备，它的作用是为连接其上的虚拟网卡提供NAT访问外网的功能，并提供DHCP服务。`virbr0`默认分配的IP是`192.168.122.1`，使用 `ifconfig` 命令查看得`virbr0`的信息如下：

```
$  ifconfig
……
 
virbr0    Link encap:Ethernet  HWaddr 52:54:00:f8:70:e3  
          inet addr:192.168.122.1  Bcast:192.168.122.255  Mask:255.255.255.0
          UP BROADCAST MULTICAST  MTU:1500  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
```

 在这种情况下，连接到virbr0上的虚拟机的虚拟网卡也是在192.168.122.0网段上的，如果让连接到virbr0上的虚拟网卡在自定义的网段上就需要修改virbr0的网段，修改方法如下： 

`# virsh  net-edit  default`

```
<network>
  <name>default</name>
  <uuid>91cc230a-bf53-487c-b296-10323705d7e8</uuid>
  <forward mode='nat'/>
  <bridge name='virbr0' stp='on' delay='0'/>
  <mac address='52:54:00:f8:70:e3'/>
  <ip address='10.0.0.1' netmask='255.255.255.0'>
    <dhcp>
      <range start='10.0.0.2' end='10.0.0.254'/>
    </dhcp>
  </ip>
</network>
```

这样就将`virbr0`的网段改成`10.0.0.0/24`，连接到`virbr0`的虚拟网卡的IP将会在`10.0.0.2/24 - 10.0.0.254/24`范围内自动分配一个。如果有需要可以自己手动给虚拟网卡配置IP并写到配置文件中去。 

# 服务器安装VNC远程桌面

因为服务器上安装的`Ubuntu 16.04 LTS  desktop`版的系统，在后续的工作中需要远程登录到服务器，虽然可以通过SSH远程管理服务器，但是可视化的界面往往会给新手用户提供很大的便利，所以可以在服务器上安装VNC。开始在服务器上安装VNC试过很多方法，VNC服务器端也有多种选择，如`VNC4server`、`tigervncserver`，感觉很麻烦，而且还不一定能安装成功，我安装的VNC服务器端是`x11VNC`，按照步骤可以很顺利地完成安装，步骤如下： 

## 安装x11VNC软件包

`$ sudo  apt-get  install  x11vnc`

## 配置访问密码

`$ sudo  x11vnc  -storepasswd  /etc/x11vnc.pass`

## 创建服务

`# vi  /lib/systemd/system/x11vnc.service      //粘贴一下代码，最后:wq 保存，请使用root用户，否则没有权限`

```
[Unit]
Description=Start x11vnc at startup.
After=multi-user.target
[Service]
Type=simple
ExecStart=/usr/bin/x11vnc -auth guess -forever -loop -noxdamage -repeat -rfbauth /etc/x11vnc.pass -rfbport 5900 -shared
[Install]
WantedBy=multi-user.target
```

## 配置防火墙，配置和启动服务

`# ufw allow 5900`

`# systemctl enable x11vnc.service`

`# systemctl daemon-reload`

`# systemctl start x11vnc.service`

完成这四个步骤然后重启就可以了。（这个VNC的安装过程可以参考http://blog.csdn.net/longhr/article/details/51657610） 

最后在你自己的电脑需要有一个vnc viewer的软件，可以在这里下载（链接：<https://pan.baidu.com/s/1o8kPqXG> 密码：v5r2） 

# 创建VM并配置相关信息

在安装好VNC后就可以登录服务器的远程桌面，打开一个terminal，在terminal中输入下面的命令可以打开Virtual Machine  Manager（注意，使用SSH远程登录服务器是无法打开virt-manager的界面的，一定要在登录了远程桌面后才能打开界面）

![VMM](https://img-blog.csdn.net/20171201233200479)

使用Virtual Machine Manager的界面可以很方便的创建虚拟机，当然也可以在命令行中使用命令创建虚拟机，这个我就不在这里说了。 

## 按照Openstack官网安装文档的主机网络配置两台虚拟机Controller和Compute

![openstack network](https://raw.githubusercontent.com/cao0507/Pictures/master/blog/openstack%20network.png)

控制节点和计算节点这两个虚拟机分别两张网卡，一张配置为桥接模式，另一张配置为NAT模式。创建虚拟机时默认是添加一张网卡的，后面可以在虚拟机的硬件信息中添加。两张虚拟网卡的配置信息如图： 

![compute1](https://raw.githubusercontent.com/cao0507/Pictures/master/blog/%E6%A1%A5%E6%8E%A5%E6%A8%A1%E5%BC%8F%E7%BD%91%E5%8D%A1.png)

上图显示的是桥接模式网卡的配置信息，Network source选择为`Bridge br0：Host device eth0` 

![NAT](https://raw.githubusercontent.com/cao0507/Pictures/master/blog/NAT%E7%BD%91%E5%8D%A1.png)

上图显示的是NAT模式网卡的配置信息，Network source选择为`Virtual network ‘default’：NAT` 

这样按照官方文档部署双节点Openstack的前期准备工作就已经做完，后面就可以按照官方文档开始安装openstack了，祝你成功。附上官方文档链接<https://docs.openstack.org/ocata/zh_CN/install-guide-ubuntu/index.html> （注：这个是在ubuntu系统下安装Ocata版本Openstack中文文档） 

---
