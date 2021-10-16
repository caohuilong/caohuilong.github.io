---
title: Hyperledger Caliper Disable TLS
date: 2018-12-15 12:54:14
categories:
  - "区块链"
tags:
  - "Hyperledger caliper"
  - "Hyperledger fabric"
---

## 前言

最近在使用 Hyperledger Caliper 时，想通过 wireshark 抓包来分析 fabric 运行流程中各阶段的数据信息，但是发现 fabric 节点间的通信使用了传输层安全（Transport Layer Security，TLS）协议，使得通信的报文的内容在抓包后无法分析。因此考虑在测试环境中暂时关闭 TLS，从而能够直接查看报文中承载的数据内容。

<!--more-->

------

## 实现过程

#### 1. 在 docker-compose 的配置文件中修改环境变量

本实验是在 Hyperledger Caliper 的测试环境中进行的，Caliper 测试工具在运行初始阶段会调用 docker-compose 启动 fabric 的网络，启动的 fabric 默认启用了 TLS，可以在其 docker-compose 的启动配置文件 docker-compose.yaml 中看到环境变量：

- `FABRIC_CA_SERVER_TLS_ENABLED=true`
- `ORDERER_GENERAL_TLS_ENABLED=true`
- `CORE_PEER_TLS_ENABLED=true`

以上三个环境变量都设置为 true。如果要 disable TLS，则需在配置文件 docker-compose.yaml 中将这三个环境变量都注释掉，或者将它们设置为 false。即：

- `FABRIC_CA_SERVER_TLS_ENABLED=false`
- `ORDERER_GENERAL_TLS_ENABLED=false`
- `CORE_PEER_TLS_ENABLED=false`

#### 2. 修改 benchmark 中的 fabric.json 文件

在 benchmark 中的每个例子中，如 simple 的网络配置文件 fabric.json 中，client 与 peer 、orderer、ca 等节点都是通过 grpcs 或 https 来通信的，而这是在使用了 TLS 时的通信方式，因此需要将其改为 grpc 或 http 来通信。修改入下：

- `orderer.url`：`grpcs://localhost:7050`  ==>  `grpc://localhost:7050`
- `ca.url`：`https://localhost:7054`  ==>  `http://localhost:7054`
- `peer1.requests`：`grpcs://localhost:7051`   ==>  `grpc://localhost:7051`
- `peer1.events`：`grpcs://localhost:7053`   ==>   `grpc://localhost:7053`

其他的都是如此修改。

> 如果用 vim 编辑器的话，可以快捷的使用全局替换功能，在 normal 模式下输入冒号：
>
> ```
> :1,$  s/grpcs/grpc/g       #表示将第一行到最后一行间的所有grpcs替换成grpc
> ```

#### 3. 错误记录

如果仅仅修改 docker-compose.yaml 文件中的环境变量，没有修改 fabric.json 中的通信方式的话，则在运行测试时会出现如下错误：

```
# create mychannel......
E1215 12:26:25.877864366    9327 ssl_transport_security.cc:989] Handshake failed with fatal error SSL_ERROR_SSL: error:1408F10B:SSL routines:SSL3_GET_RECORD:wrong version number.
E1215 12:26:25.879670836    9327 ssl_transport_security.cc:989] Handshake failed with fatal error SSL_ERROR_SSL: error:1408F10B:SSL routines:SSL3_GET_RECORD:wrong version number.
error: [Orderer.js]: sendBroadcast - on error: "Error: 14 UNAVAILABLE: Connect Failed\n    at createStatusError (/home/user1/caliper/node_modules/grpc/src/client.js:64:15)\n    at ClientDuplexStream._emitStatusIfDone (/home/user1/caliper/node_modules/grpc/src/client.js:270:19)\n    at ClientDuplexStream._readsDone (/home/user1/caliper/node_modules/grpc/src/client.js:236:8)\n    at readCallback (/home/user1/caliper/node_modules/grpc/src/client.js:296:12)"
not ok 1 Failed to create channels Error: SERVICE_UNAVAILABLE at ClientDuplexStream.<anonymous> (/home/user1/caliper/node_modules/fabric-client/lib/Orderer.js:136:21) at emitOne (events.js:116:13) at ClientDuplexStream.emit (events.js:211:7) at ClientDuplexStream._emitStatusIfDone (/home/user1/caliper/node_modules/grpc/src/client.js:271:12) at ClientDuplexStream._readsDone (/home/user1/caliper/node_modules/grpc/src/client.js:236:8) at readCallback (/home/user1/caliper/node_modules/grpc/src/client.js:296:12)
  ---
    operator: fail
    at: channels.reduce.then.then.catch (/home/user1/caliper/src/fabric/create-channel.js:159:19)
    stack: |-
      Error: Failed to create channels Error: SERVICE_UNAVAILABLE
          at ClientDuplexStream.<anonymous> (/home/user1/caliper/node_modules/fabric-client/lib/Orderer.js:136:21)
          at emitOne (events.js:116:13)
          at ClientDuplexStream.emit (events.js:211:7)
          at ClientDuplexStream._emitStatusIfDone (/home/user1/caliper/node_modules/grpc/src/client.js:271:12)
          at ClientDuplexStream._readsDone (/home/user1/caliper/node_modules/grpc/src/client.js:236:8)
          at readCallback (/home/user1/caliper/node_modules/grpc/src/client.js:296:12)
          at Test.assert [as _assert] (/home/user1/caliper/node_modules/tape/lib/test.js:224:54)
          at Test.bound [as _assert] (/home/user1/caliper/node_modules/tape/lib/test.js:76:32)
          at Test.fail (/home/user1/caliper/node_modules/tape/lib/test.js:317:10)
          at Test.bound [as fail] (/home/user1/caliper/node_modules/tape/lib/test.js:76:32)
          at channels.reduce.then.then.catch (/home/user1/caliper/src/fabric/create-channel.js:159:19)
          at <anonymous>
          at process._tickCallback (internal/process/next_tick.js:189:7)
  ...
fabric.init() failed, Error: Fabric: Create channel failed
    at channels.reduce.then.then.catch (/home/user1/caliper/src/fabric/create-channel.js:160:31)
    at <anonymous>
    at process._tickCallback (internal/process/next_tick.js:189:7)
[Transaction Info] - Submitted: 0 Succ: 0 Fail:0 Unfinished:0
unexpected error, Error: Fabric: Create channel failed
    at channels.reduce.then.then.catch (/home/user1/caliper/src/fabric/create-channel.js:160:31)
    at <anonymous>
    at process._tickCallback (internal/process/next_tick.js:189:7)
```

这个问题需要注意。

------

## 结果

在 TLS 被开启或关闭两种情况下，能够发现关闭 TLS 后，系统的吞吐率略有提升，这是可想而知的，毕竟减少了一层传输层安全协议的封装。结果图如下：

- **Enable TLS**

  ![](https://github.com/cao0507/My-Pictures-Repository/blob/master/Blockchain/caliper/enable_tls.png?raw=true)

- **Disable TLS**

  ![](https://github.com/cao0507/My-Pictures-Repository/blob/master/Blockchain/caliper/disable_tls.png?raw=true)

不过以上的结果在实际中的意义并不大，因为在实际应用中肯定需要进行传输层安全协议的封装，不然这区块链的安全从和谈起。

如下图可以看到关闭 TLS 后，抓包后能够查看数据内容：

![](https://github.com/cao0507/My-Pictures-Repository/blob/master/Blockchain/caliper/disable-tls%20%E6%8A%93%E5%8C%85%E7%BB%93%E6%9E%9C.png?raw=true)

太不安全了！所以以上内容均只能应用于测试。

------