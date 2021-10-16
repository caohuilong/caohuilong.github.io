---
title: Openstack环境下手动安装Tacker
date: 2017-12-23 14:12:37
categories:
  - "Openstack"
tags:
  - "Openstack"
  - "Tacker"
---

# 概述
本文参考官方文档，在现有的openstack平台上，手动安装Tacker。基础的openstack平台包含了最核心的keystone、glance、nova、neutron、horizon这5个组件，但是Tacker还需要预先安装好Mistral和Barbican这两个组件，在安装好这两个组件后就可以开始按照以下步骤安装Tacker了。
<!--more-->
参考：[*官方文档链接*](https://docs.openstack.org/tacker/latest/install/manual_installation.html)

**注：** 本文涉及到的密码都统一设置成*openstack*。

# 一、安装Tacker server
## 1、创建数据库
```
# mysql
MariaDB [(none)]> CREATE DATABASE tacker;
MariaDB [tacker]> GRANT ALL PRIVILEGES ON tacker.* TO 'tacker'@'localhost' IDENTIFIED BY 'openstack';
MariaDB [tacker]> GRANT ALL PRIVILEGES ON tacker.* TO 'tacker'@'%' IDENTIFIED BY 'openstack';
MariaDB [tacker]> exit;
```
## 2、创建user、role、endpoints
### 1)获得admin凭证
    # . admin-openrc

### 2)创建tacker用户，密码为openstack
    # openstack user create --domain default --password openstack tacker

### 3)给tacker用户赋予admin权限
    # openstack role add --project service --user tacker admin

### 4)创建tacker服务
    # openstack service create --name tacker \
        --description "Tacker Project" nfv-orchestration

### 5)创建endpoints
```
# openstack endpoint create --region RegionOne nfv-orchestration \
           public http://controller:9890/
# openstack endpoint create --region RegionOne nfv-orchestration \
           internal http://controller:9890/
# openstack endpoint create --region RegionOne nfv-orchestration \
           admin http://controller:9890/
```

## 3、下载Tacker源码
    # git clone https://github.com/openstack/tacker -b stable/ocata
## 4、安装Tacker环境依赖包
    # cd tacker
    # pip install -r requirements.txt

## 5、安装Tacker
    # python setup.py install

## 6、创建Tacker日志文件夹
    # mkdir -p /var/log/tacker

## 7、生成配置文件
    # ./tools/generate_config_file_sample.sh
    这时生成的配置文件在etc/tacker/tacker.conf.sample，需要将其重命名为tacker.conf
    # mv etc/tacker/tacker.conf.sample  etc/tacker/tacker.conf
## 8、修改配置文件
```
# vi etc/tacker/tacker.conf

[DEFAULT]
auth_strategy = keystone
policy_file = /usr/local/etc/tacker/policy.json
debug = True
use_syslog = False
bind_host = 10.0.0.11
bind_port = 9890
service_plugins = nfvo,vnfm

state_path = /var/lib/tacker
...

[nfvo]
vim_drivers = openstack

[keystone_authtoken]
memcached_servers = 11211
region_name = RegionOne
auth_type = password
project_domain_name = Default
user_domain_name = Default
username = tacker
project_name = service
password = openstack
auth_url = http://controller:35357
auth_uri = http://controller:5000
...

[agent]
root_helper = sudo /usr/local/bin/tacker-rootwrap /usr/local/etc/tacker/rootwrap.conf

[database]
connection = mysql://tacker:openstack@controller:3306/tacker?charset=utf8

[tacker]
monitor_driver = ping,http_ping
```
## 9、复制配置文件到配置文件夹
    # cp etc/tacker/tacker.conf  /usr/local/etc/tacker/

## 10、初始化数据库信息
    # /usr/local/bin/tacker-db-manage --config-file /usr/local/etc/tacker/tacker.conf upgrade head

# 二、安装Tacker client
## 1、下载Tacker-client源码
    # git clone https://github.com/openstack/python-tackerclient -b stable/ocata
## 2、安装Tacker-client模块
    # cd python-tackerclient
    # python setup.py install

# 三、安装Tacker horizon
## 1、下载Tacker-horizon源码
    # git clone https://github.com/openstack/tacker-horizon -b stable/ocata
## 2、安装Tacker-horizon模块
    # cd tacker-horizon
    # python setup.py install
安装好tacker-horizon后，admin用户登录dashboard界面就可以看到Tacker相关的VNFM和NFVO，如图：
![tacker-horizon](https://raw.githubusercontent.com/cao0507/Pictures/master/blog/tacker%20horizon.png)


# 四、开启Tacker server
打开一个新的终端，开启Tacker-server，因为Tacker-server的程序会独占这个终端。
```
sudo python /usr/local/bin/tacker-server \
    --config-file /usr/local/etc/tacker/tacker.conf \
    --log-file /var/log/tacker/tacker.log
```

---


**需注意的一个问题：**

在安装完Tacker而没有装Mistral时创建VIM的结果如下：
```
root@controller:/home/openstack# tacker vim-register --is-default --config-file config.yaml test_vim
The resource could not be found.

或者是这种错误：Expecting to find domain in project. The server could not comply with the request since it is either malformed or otherwise incorrect. The client is assumed to be in error.
```
经过查阅资料，知道这个问题是因为Tacker在创建VIM时要调用Mistral而造成的。所以在使用tacker之前需要先安装好Mistral（可以在安装tacker前安装Mistral，也可以在tacker安装之后安装Mistral，后续还需继续了解）。

---
