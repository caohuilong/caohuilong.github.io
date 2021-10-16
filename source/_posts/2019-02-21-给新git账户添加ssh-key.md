---
title: 给新 Git 账户添加 ssh-key
date: 2019-02-21 17:02:23
categories:
  - "Linux"
tags:
  - "Git"
---

## 背景

在使用 git 的时候，git 与远程服务器一般通过 https 进行传输，这种传输方式在每次 push 和 pull 时都需要输入账户和密码，比较麻烦。所以更好的方法是通过 ssh 进行传输，这需要在本机上创建 ssh-key 密钥对，并把其中的公钥添加到远程的 Git 服务器中。但有时又会使用到多个 git 账户登录不同的 git 服务器，所以就涉及到添加 ssh-key 密钥对了。

<!--more-->

我的环境中最初是针对 GitHub 的账户设置了 ssh 的密钥对，然后我现在需要针对另一个 git 账户进行设置密钥对，比如牛客网的 git 服务器。

## 添加操作过程

**1. 新建 SSH key：**

```
$ cd ~/.ssh
$ ssh-keygen -t rsa -C "xxx@email.com"
Generating public/private rsa key pair.
Enter file in which to save the key (/home/chl/.ssh/id_rsa): id_rsa_nowcoder
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in id_rsa_nowcoder.
Your public key has been saved in id_rsa_nowcoder.pub.
The key fingerprint is:
SHA256:hjOHuHTiXqCjTPCBf81owybsdflq/HVWcq4jBZjnCNs xxx@email.com
The key's randomart image is:
+---[RSA 2048]----+
|                 |
|   . o           |
|o   Oo o         |
|=...oBoo         |
| =.+*+O.S o      |
|+.o=EB.=.=       |
|o.* +.o .o .     |
|.o + o.o.        |
|o.  o  ...       |
+----[SHA256]-----+
```

> 上面设置名称为 id_rsa_nowcoder

**2. 新秘钥添加到 SSH agent 中**

因为默认只读取 id_rsa，为了让 SSH 识别新的私钥，需将其添加到 SSH agent 中：

```
$ ssh-add ~/.ssh/id_rsa_nowcoder
Could not open a connection to your authentication agent.
```

但是出现了 Could not open a connection to your authentication agent. 的错误，用一下方法解决：

```
$ ssh-agent bash
$ ssh-add ~/.ssh/id_rsa_nowcoder
Identity added: ~/.ssh/id_rsa_nowcoder (~/.ssh/id_rsa_nowcoder)
```

**3. 在 git 账户中添加 SSH key**

登录 git 账户中添加，完成之后，SSH key 就生效了。检测方法：

```
$ git clone 你的仓库ssh地址
```

若这时不再询问密码，说明设置生效。

**4. 更改远程仓库地址**

- 修改命令：

  ```
  $ git remote set-url origin 你的仓库ssh地址
  ```

- 或者先删后加：

  ```
  $ git remote rm origin
  $ git remote add origin 你的仓库ssh地址
  ```

- 或者修改 config 文件

  ```
  $ git config -e
  修改 url
  ```

更改完成之后，再通过 git push 或 git pull 就不需要输入账号和密码了。

------

