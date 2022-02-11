---
title: 如何寻找veth pair的peer设备
date: 2021-04-21 22:17:53
categories:
  - "网络虚拟化"
tags:
  - "veth pair"
  - "namespace"
---

在网络虚拟化环境中，veth pair常被用跨network namespace的通信，Docker容器的网络就是基于veth pair与network namespace来实现的通，常将veth pair一端置于容器中，另一端连接到虚拟网桥上，从而实现容器网络。但是在网络调试时，经常会需要在veth pair设备上面抓包，在一些虚拟网络设备较多的场景下，不太容易肉眼观察容器中的虚拟网卡与宿主机的虚拟网桥连接的veth pair设备。因此需要方法来寻找veth pair的peer设备。

<!--more-->

这里通过创建一个network namespace来模拟一个容器，查找添加到network namespace的veth pair在宿主机的根namespace中的peer设备。

##### 1、创建虚拟网卡对

```shell
# ip link add veth1 type veth peer name veth2
# ip link set veth1 up
# ip link set veth2 up
```

##### 2、创建network namespace，并添加虚拟网卡

```shell
# ip netns add ns1
# ip link set dev veth1 netns ns1
```

##### 3、在不同命名空间中查看veth-pair的peer设备

- 方法一

  在目标命名空间中：

  ```shell
  # ip netns exec ns1 cat /sys/class/net/veth1/iflink
  7
  ```

  然后，在主机上遍历/sys/claas/net下面的全部目录，查看子目录ifindex的值和容器里查出来的iflink值相当的veth名字，这样就找到了容器和主机的veth pair关系。例如，下面的例子中主机上veth2的ifindex刚好是7，意味着是目标容器veth pair的另一端。

  ```shell
  # cat /sys/class/net/veth2/ifindex 
  7
  ```

- 方法二

  在目标命名空间中查看：

  ```shell
  # ip netns exec ns1 ip link list
  1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT group default qlen 1000
      link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
  8: veth1@if7: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
      link/ether c2:d7:5c:a1:d1:e2 brd ff:ff:ff:ff:ff:ff link-netnsid 0
  ```

  从上面的命令输出可以看到8: veth1@if7，其中8是veth1接口的index，7是和它成对的veth的index。当host执行下面的命令时，可以看到对应7的veth网卡是哪一个，这样就得到了容器和veth pair的关系。

  ```shell
  # ip link list
  7: veth2@if8: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
      link/ether da:5e:1a:ea:e6:80 brd ff:ff:ff:ff:ff:ff link-netnsid 0
  ```

- 方法三

  可以通过ethtool-S命令列出veth pair对端的网卡index，例如：

  ```shell
  # ip netns exec ns1 ethtool -S veth1
  NIC statistics:
       peer_ifindex: 7
  ```

  而主机上index为6的网卡为：

  ```shell
  # ip addr
  7: veth2@if8: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
      link/ether da:5e:1a:ea:e6:80 brd ff:ff:ff:ff:ff:ff link-netnsid 0
  ```

  