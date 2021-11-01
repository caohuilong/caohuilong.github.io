---
title: CentOS6.5安装cloudinit-18
date: 2020-12-09 16:40:19
categories:
 - "Linux"
tags:
 - "CentOS"
 - "Cloud-init"
 - "镜像"
---

目前有个 CentOS6.x 的系统镜像，默认支持 cloud-init-0.7.5 版本 rpm，但是 cloud-init-0.7.5 无法通过本地化 QEMU 利用 NoCloud 数据源进行 IP 地址管理，因此无法实现虚机的网络初始化，需要手动进行配置。经过测试实验，cloud-init 17 以上版本能够支持 NoCloud 数据源信息注入，但是 cloud-init 17 及以上版本需要 python2.7 或者 python3 才能运行，而 CentOS6.x 系统默认使用 python2.6。因此为了在 CentOS6.x 上安装 cloud-init 17 以上的版本，需要先升级 python2.7，然后再安装 cloud-init。

<!--more-->

---

# 具体步骤

### 更新软件包

##### 1、安装 EPEL 源

```
yum install -y epel-release
```

> - 如果安装软件出现问题，见后续的问题记录1。
>
> - 此外，安装 EPEL 源之后，安装软件还可能出问题，继续见后续问题记录2。

##### 2、安装 cloud-init 等软件包

```
yum -y install qemu-guest-agent cloud-utils-growpart gdisk libicu cloud-init dracut-modules-growroot
```

##### 3、安装 versionlock 并锁定 cloud-init

```
yum -y install yum-plugin-versionlock
yum versionlock add cloud-init
```

##### 4、修改配置

```
rpm -qa kernel | sed 's/^kernel-//'  | xargs -I {} dracut -f /boot/initramfs-{}.img {}

chkconfig cloud-config on
chkconfig cloud-init on
chkconfig cloud-final on

sed -i '/$cloud_init $CLOUDINITARGS init/a \
	    service network restart' /etc/init.d/cloud-init
```

### 安装 python2.7

##### 1、源码安装 python

```
yum -y install gcc openssl-devel bzip2-devel
cd /usr/local/src
wget https://www.python.org/ftp/python/2.7.16/Python-2.7.16.tgz
tar xzf Python-2.7.16.tgz
cd Python-2.7.16
./configure --enable-optimizations
make altinstall
curl "https://bootstrap.pypa.io/get-pip.py" -o "get-pip.py"
python2.7 get-pip.py
```

> 下载 get-pip.py 脚本时可能碰到 curl: (35) SSL connect error 的问题，见问题记录3。

##### 2、下载 cloud-init 依赖的 python 包

```
pip2.7 install six
pip2.7 install pyyaml
pip2.7 install requests
pip2.7 install jsonpatch
pip2.7 install configobj
```

> pip 下载失败可以更换国内源，见问题记录4。

### 更新 Cloud-init

##### 1、下载安装 cloud-init

```
cd /usr/local/src
mkdir cloud-init
cd cloud-init
wget http://ftp.riken.jp/Linux/cern/centos/7/updates/x86_64/Packages/cloud-init-18.5-3.el7.centos.x86_64.rpm
rpm2cpio cloud-init-18.5-3.el7.centos.x86_64.rpm  | cpio -idmv
mv /usr/local/src/cloud-init/etc/bash_completion.d/* /etc/bash_completion.d
rm -rf /etc/cloud/
mv /usr/local/src/cloud-init/etc/cloud  /etc/
mv -f /usr/local/src/cloud-init/etc/rsyslog.d/* /etc/rsyslog.d/
mkdir -p /run/cloud-init
sed -i 's/\/usr\/bin\/python/\/usr\/local\/bin\/python2.7/g' /usr/local/src/cloud-init/usr/bin/*
mv -f /usr/local/src/cloud-init/usr/bin/* /usr/bin/
mv /usr/local/src/cloud-init/usr/lib/python2.7/site-packages/* /usr/local/lib/python2.7/site-packages/
mkdir -p /usr/lib/tmpfiles.d
mv /usr/local/src/cloud-init/usr/lib/tmpfiles.d/* /usr/lib/tmpfiles.d/
rm -rf /usr/libexec/cloud-init
mv /usr/local/src/cloud-init/usr/libexec/* /usr/libexec/
rm -rf /usr/local/src/*
```

##### 2、修改 cloud-init 配置

```
sed -i "s/disable_root.*/disable_root\: 0/;s/ssh_pwauth.*/ssh_pwauth\: 0/" /etc/cloud/cloud.cfg
```

最后执行验证 cloud-init 更新成功：

```
# cloud-init
/usr/bin/cloud-init 18.5
```

大功告成！！！后面就可以对虚机做一些别的配置，最后制作成一个镜像了。

---

# 可能遇到的问题记录

##### 1、yum 下载软件包出错 Error: Cannot find a valid baseurl for repo: base

```
# yum install -y epel-release
Loaded plugins: fastestmirror
Determining fastest mirrors
YumRepo Error: All mirror URLs are not using ftp, http[s] or file.
 Eg. Invalid release/repo/arch combination/
removing mirrorlist with no valid mirrors: /var/cache/yum/x86_64/6/base/mirrorlist.txt
Error: Cannot find a valid baseurl for repo: base
```

**解决方法**：修改 yum 源，更换成国内源

```
cp /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.bak
curl http://mirrors.163.com/.help/CentOS5-Base-163.repo -o /etc/yum.repos.d/CentOS-Base.repo
```

使用 yum 再次安装又出现以下错误：

```
# yum install -y epel-release
Loaded plugins: fastestmirror
Loading mirror speeds from cached hostfile
http://mirrors.163.com/centos/6/addons/x86_64/repodata/repomd.xml: [Errno 14] PYCURL ERROR 22 - "The requested URL returned error: 404 Not Found"
Trying other mirror.
Error: Cannot retrieve repository metadata (repomd.xml) for repository: addons. Please verify its path and try again
```

再次搜索之后，发现可以使用 **vault.centos.org** 作为更新源

**具体操作步骤：**

1. 关闭 fastestmirror

   ```
   # vi /etc/yum/pluginconf.d/fastestmirror.conf
   #修改参数
   enable=0
   ```

2. 备份原来的源

   ```
   # mv /etc/yum.repo.d/CentOS-Base.repo /etc/yum.repo.d/CentOS-Base.repo.bak
   ```

3. 更换 yum 源

   - 阿里云 Vault 源

     ```
     # curl -o /etc/yum.repos.d/CentOS-Base.repo https://www.xmpan.com/Centos-6-Vault-Aliyun.repo 
     
     # wget -O /etc/yum.repos.d/CentOS-Base.repo https://static.lty.fun/%E5%85%B6%E4%BB%96%E8%B5%84%E6%BA%90/SourcesList/Centos-6-Vault-Aliyun.repo
     ```

     > 我的虚机上面没有 wget 命令，所以用 curl 命令来下载文件。

   - 官方 Vault 源

     ```
     # wget -O /etc/yum.repos.d/CentOS-Base.repo https://static.lty.fun/%E5%85%B6%E4%BB%96%E8%B5%84%E6%BA%90/SourcesList/Centos-6-Vault-Official.repo
     ```

4. 清理并构建缓存

   ```
   yum clean all
   yum makecache
   yum repolist
   ```

再次使用 yum 安装软件，成功。

---

##### 2、安装 EPEL 源后，使用报错

具体错误如下：

```
# yum install cloud-init
Error: Cannot retrieve repository metadata (repomd.xml) for repository: epel. Please verify its path and try again
```

**解决方法：**修改 /etc/yum.repo.d/epel.repo，将 mirrorlist=https://xxxx 修改为 http，即可正常使用。

---

##### 3、curl: (35) SSL connect error 问题处理

下载 get-pip.py 脚本出现下面错误：

```
# curl https://bootstrap.pypa.io/get-pip.py -o get-pip.py
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:--  0:00:03 --:--:--     0
curl: (35) SSL connect error
```

解决方法：无法在服务器使用 curl 命令访问 https 域名，原因是 nss 版本有点旧了，`yum -y update nss` 更新一下，重新 curl 即可！

```
yum -y update nss
```

---

##### 4、更换 pip 源

在 `~/.pip/pip.conf` 文件中添加或修改：

```
# mkdir ~/.pip
# cat ~/.pip/pip.conf
[global]
index-url = http://mirrors.aliyun.com/pypi/simple/

[install]
trusted-host=mirrors.aliyun.com
```

