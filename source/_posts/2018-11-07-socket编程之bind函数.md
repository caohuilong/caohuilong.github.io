---
title: socket编程之bind函数
date: 2018-11-07 21:41:09
categories:
  - "UNIX网络编程"
tags:
  - "bind"
---

### bind 函数：关联地址和套接字

定义：

```cpp
#include <sys/socket.h>
int bind(int sockfd, const struct sockaddr *addr, socklen_t len);
```

返回值：若成功，返回 0；若出错，返回 -1.

<!--more-->

### 使用 bind 时遇到的错误

在练习 UNP 代码 daytimetcpsrv.c 时遇到两个问题：

1. **Permission denied**

   这是因为地址中的端口号必须不小于 1024，除非该进程具有相应的特权（即 root 用户）。

2. **Address already in use**

   这个问题有时会让人很疑问，明明已经结束了使用对应端口的进程，端口应该不是 `in use` 的啊，但却无法再次调用 bind 函数来绑定该端口到一个套接字端点（bind 函数返回 `EADDRINUSE`）。其实这是由 TCP 套接字状态 `TIME_WAIT` 引起的，该状态在套接字关闭后约保留 2 到 4 分钟，因此无法再次绑定刚刚使用的端口。在 `TIME_WAIT` 状态退出之后，套接字被删除，该地址才能被重新绑定而不出问题。

   可以通过 netstat -ant 来查看这个端口还处于 `TIME_WAIT` 状态：

   等待 `TIME_WAIT` 结束可能是令人恼火的一件事，特别是如果您正在开发一个套接字服务器，就需要停止服务器来做一些改动，然后重启。幸运的是，有方法可以避开 `TIME_WAIT` 状态。可以给套接字应用 `SO_REUSEADDR` 套接字选项，以便端口可以马上重用。

   对于 daytimetcpsrv.c，可以加上以下代码：

   ```cpp
   int reuse = 1;
   listenfd = Socket(AF_INET, SOCK_STREAM, 0);
   if (setsockopt(listenfd, SOL_SOCKET, SO_REUSEADDR, &reuse, sizeof(reuse)) < 0)
    {
     perror("setsockopet error\n");
     return -1;
    }
   ```

   重新编译运行。

### 关于 TIME_WAIT 状态

TCP 设计中之所以要让一个旧的连接处于 TIME_WAIT 状态是因为要防止旧连接的老的重复分组出现在新连接中。拿 UNP 上面的例子来说：

假设在 12.106.32.245 的端口 1500 和 206.168.112.219 的端口 21 之间有一个 TCP 连接。当我们关闭这个连接后，很快又重新建立一条相同 IP 和端口的 TCP 连接。在这种情况下，假如旧连接在网络中还存在没有被丢弃的重复分组，而且重复分组又出现在了新连接中了，TCP 将无法正确处理这个分组。为了防止这种情况的发生，TCP 将刚关闭的连接置于 TIME_WAIT 状态，不允许给处于该状态的连接启动新的化身，持续时间是 2MSL，如此将能保证该连接的老的重复分组都已在网络中消逝。

> 注：是主动执行关闭 TCP 连接的那端将处于 TIME_WAIT 状态。

------

