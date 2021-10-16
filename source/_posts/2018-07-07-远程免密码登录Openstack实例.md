---
title: 远程免密码登录Openstack实例
date: 2018-07-07 22:15:41
categories:
  - "Openstack"
tags:
  - "Openstack"
  - "Namespace"
  - "SSH"
---
# 背景

一般情况下，可以通过`Openstack Dashboard`的控制台来访问用户创建的实例`Instance`，对于管理员来说，通过这种方法来访问会觉得很不方便，因为每次都需要打开浏览器来输入网址，每次点击都需要等待响应，登录到实例后控制台的响应也不是很及时且有卡顿。因此，本文介绍如何通过命名空间来实现免密码登录Openstack实例。

<!--more-->

# 命名空间

在Linux中，网络命名空间可以被认为是隔离的拥有单独网络栈（网卡、路由转发表、iptables）的环境。网络命名空间经常用来隔离网络设备和服务，只有拥有同样网络命名空间的设备，才能看到彼此。openstack中就采用命名空间来实现不同网络的隔离。

- 使用`ip netns`来查看已经存在的命名空间：

  ```
  $ ip netns
  qrouter-e94975e8-4688-4858-8f64-86a18eea81ed
  qdhcp-34ba192f-ceea-4c86-addc-a5d14c6a34a8
  ```

  `qdhcp`开头的名字空间是dhcp服务器使用的，`qrouter`开头的则是router服务使用的。 

- 查看openstack的网络：

  ```
  $ openstack network list
  +--------------------------------------+---------+--------------------------------------+
  | ID                                   | Name    | Subnets                              |
  +--------------------------------------+---------+--------------------------------------+
  | 34ba192f-ceea-4c86-addc-a5d14c6a34a8 | private | 74dbc6f4-ae59-4af5-b941-1f4d04918607 |
  | fc9b502a-d472-46f5-8570-b0d3915759cf | public  | de903618-fb31-412a-b46d-6ab593985b03 |
  +--------------------------------------+---------+--------------------------------------+
  ```

  可以看到`private`网络的dhcp服务器对应的命名空间`qdhcp-34ba192f-ceea-4c86-addc-a5d14c6a34a8`的名字中包含了`private`网络的ID。而本次测试的远程实例就是创建在private网络下的。

- 通过 `ip netns exec namespace_id command` 来在指定的网络名字空间中执行网络命令，记得加上`sudo`权限，例如 

  ```
  $ sudo ip netns exec qdhcp-34ba192f-ceea-4c86-addc-a5d14c6a34a8 ifconfig
  lo        Link encap:Local Loopback  
            inet addr:127.0.0.1  Mask:255.0.0.0
            inet6 addr: ::1/128 Scope:Host
            UP LOOPBACK RUNNING  MTU:65536  Metric:1
            RX packets:190 errors:0 dropped:0 overruns:0 frame:0
            TX packets:190 errors:0 dropped:0 overruns:0 carrier:0
            collisions:0 txqueuelen:1 
            RX bytes:65824 (65.8 KB)  TX bytes:65824 (65.8 KB)
  
  tap4e2f68bb-84 Link encap:Ethernet  HWaddr fa:16:3e:32:19:6b  
            inet addr:10.0.0.2  Bcast:10.0.0.63  Mask:255.255.255.192
            inet6 addr: fe80::f816:3eff:fe32:196b/64 Scope:Link
            UP BROADCAST RUNNING MULTICAST  MTU:1450  Metric:1
            RX packets:110210 errors:0 dropped:0 overruns:0 frame:0
            TX packets:107493 errors:0 dropped:0 overruns:0 carrier:0
            collisions:0 txqueuelen:1 
            RX bytes:21335986 (21.3 MB)  TX bytes:8426397 (8.4 MB)
  ```

- ssh远程登录实例：

  `sudo ip netns exec namespace_id ssh $username@ip`

  通过该命令来实现从控制节点通过ssh服务远程访问Openstack实例。

# 发送公共秘钥

要实现免密码远程登录实例，首先需要将控制节点`root用户`的ssh的公共秘钥发送到远程实例，也就是`/root/.ssh/id_rsa.pub`文件中的内容，远程实例收到后会将公共密钥保存到登录用户的`.ssh/authorized_keys`文件中，这样下次登录远程实例时就不再需要密码。

- 发送公共密钥：

  ```
  $ sudo ip netns exec qdhcp-34ba192f-ceea-4c86-addc-a5d14c6a34a8 ssh-copy-id -i /root/.ssh/id_rsa.pub openstack@10.0.0.9
  
  /usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "/root/.ssh/id_rsa.pub"
  /usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
  /usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
  openstack@10.0.0.9's password: 
  
  Number of key(s) added: 1
  
  Now try logging into the machine, with:   "ssh 'openstack@10.0.0.9'"
  and check to make sure that only the key(s) you wanted were added.
  ```

  **注意：**发送的公共秘钥必须是控制节点`root`用户的，因为在进入命名空间执行命令时需要加上`sudo`权限，而`sudo`是用来以其他身份来执行命令的，预设的身份为`root`，这样在`ssh`登录远程实例时是以控制节点的`root`用户来登录远程实例的`openstack`用户，因此需要将控制节点`root`用户的公共密钥发送给远程实例，root用户的公共密钥的路径是`/root/.ssh/id_rsa.pub`。

- 查看远程实例的`authorized_keys`：

  ```
  $ vi ~/.ssh/authorized_keys
  ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCjXQQUtIcLLcvBXVudZDBbQFK8BT/hB67oOrs792sfCuMhxvxvFRbma5UmnxwxOhXUIRjdz4u7tWhR3VVhqqnlHGDKOQVje/t2QtTlXXcBI3kGnc0Epem2NRMgRKp/h/Y1EOwtPNHRDVfr8C2znilXpWW1ueigHuJF4TWT7vEjgbApmWhopZcOXKbLkSu5dxLGUO3TzGqkASgpLG2XyuUJVqoREr5wbAZytq7R2p5KCxUZ6T7sDUQG+xmFPsfPg3MUHQmatTvtSf+mImotTkNSqOp2Itct9afX7SPkRncrXVWJ0qutbrRrkjRJm1l/sCjFBOD0x6txcFBX30nPvkDx root@controller
  ```

- 查看控制节点`root`用户公共密钥：

  ```
  # vi /root/.ssh/id_rsa.pub
  ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCjXQQUtIcLLcvBXVudZDBbQFK8BT/hB67oOrs792sfCuMhxvxvFRbma5UmnxwxOhXUIRjdz4u7tWhR3VVhqqnlHGDKOQVje/t2QtTlXXcBI3kGnc0Epem2NRMgRKp/h/Y1EOwtPNHRDVfr8C2znilXpWW1ueigHuJF4TWT7vEjgbApmWhopZcOXKbLkSu5dxLGUO3TzGqkASgpLG2XyuUJVqoREr5wbAZytq7R2p5KCxUZ6T7sDUQG+xmFPsfPg3MUHQmatTvtSf+mImotTkNSqOp2Itct9afX7SPkRncrXVWJ0qutbrRrkjRJm1l/sCjFBOD0x6txcFBX30nPvkDx root@controller
  ```

  可以看到，两者是一样的，说明控制节点的`root`用户已经被授权通过公共密钥来访问远程实例。

# 免密码登录

- 免密码登录远程实例：

  ```
  $ sudo ip netns exec qdhcp-34ba192f-ceea-4c86-addc-a5d14c6a34a8 ssh openstack@10.0.0.9
  ```

  这样就通过`ssh`服务免密码远程登录`Openstack`实例，而不需要通过`Dashboard`的控制台来登录实例。

---
