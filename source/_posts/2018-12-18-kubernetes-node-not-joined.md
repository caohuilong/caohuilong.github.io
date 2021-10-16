---
title: kubeadm join success but node not joined
date: 2018-12-18 16:07:53
categories:
  - "Kubernetes"
tags:
  - "kubeadm"
  - "Bugs"
---

## 问题

在搭建 Kubernetes 集群时，遇到这样一个问题，就是在 node 节点上使用 kubeadm join 时能够成功的加入节点，但是在 master 节点上却无法查看集群中的 node 节点。如下：

<!--more-->

```
node1$ sudo kubeadm join --token 1bc310.cb323487a828849e 10.2.7.114:6443 --discovery-token-ca-cert-hash sha256:3d39f8fe34a043ccef4821014fd6d3e0f222614d37d59a6e4944c74f257c6d4d
[preflight] Running pre-flight checks.
	[WARNING FileExisting-crictl]: crictl not found in system path
[discovery] Trying to connect to API Server "10.2.7.114:6443"
[discovery] Created cluster-info discovery client, requesting info from "https://10.2.7.114:6443"
[discovery] Requesting info from "https://10.2.7.114:6443" again to validate TLS against the pinned public key
[discovery] Cluster info signature and contents are valid and TLS certificate validates against pinned roots, will use API Server "10.2.7.114:6443"
[discovery] Successfully established connection with API Server "10.2.7.114:6443"

This node has joined the cluster:
* Certificate signing request was sent to master and a response
  was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the master to see this node join the cluster.
```

```
master$ kubectl get nodes
NAME      STATUS    ROLES     AGE       VERSION
ubuntu    Ready     master    4h        v1.9.1
```

## 解决方法

出现这个问题的原因是 node 节点的主机名与 master 节点的相同，因此需要给 所有的 node 节点取与 master 节点不同的主机名。

修改 `/etc/hostname` 以及 `/etc/hosts` 文件中的主机名，再通过命令临时设置主机名：`sudo hostname 主机名`。

配置完之后，在 node 节点上执行 `kubeadm reset`， 再重新执行 `kubeadm join` 。

最后在 master 节点上查看节点：

```
$ kubectl get nodes
NAME      STATUS    ROLES     AGE       VERSION
node1     Ready     <none>    1h        v1.9.1
node2     Ready     <none>    1h        v1.9.1
ubuntu    Ready     master    6h        v1.9.1
```

可以看到有两个 node 节点。

参考：[issue 61224](https://github.com/kubernetes/kubernetes/issues/61224)

---

