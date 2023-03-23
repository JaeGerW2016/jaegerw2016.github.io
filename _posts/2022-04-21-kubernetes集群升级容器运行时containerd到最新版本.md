---
layout:     post
title:     Kubernetes集群升级容器运行时containerd到最新版本
subtitle:  CVE-2022-23648 –来自 containerd CRI 启动的容器的任意主机文件访问及其对 Kubernetes 的影响
date:       2022-04-21
author:     J
catalog: true
tags:
    - Containerd
    - Kubernetes
---

# 背景

最近爆出的漏洞 - [CVE-2022-23648](https://nvd.nist.gov/vuln/detail/CVE-2022-23648) - 在的容器运行时 containerd 中，特别是允许容器从主机获得对文件的只读访问权限。虽然一般容器隔离可以阻止此类访问，但在[Kubernetes](https://www.armosec.io/glossary/kubernetes/)中，它尤其危险，因为众所周知且高度敏感的文件存储在主机上的已知位置。

### 对 Kubernetes 的影响

最吸引攻击者攻击的文件将是 kubelet 的私钥，用于在节点 kubelet 和 KubeAPI 服务器之间进行通信。Kubelet 是 Kubernetes 中绝对可信的组件，它可以要求 KubeAPI 服务器提供存储在 ETCD 中的任何信息，例如[Kubernetes Secrets](https://www.armosec.io/blog/revealing-the-secrets-of-kubernetes-secrets/)、 ConfigMaps 等。这些信息可以被泄露或在本地用于访问受保护的接口和数据资产。 Kubernetes 甚至整个云基础设施。

### 您的集群是否容易受到攻击？

如果您运行 1.6.1、1.5.10 和 1.14.12 之前的 containerd 版本，您的集群很容易受到利用此漏洞的潜在攻击。

### 缓解和验证

如果您的集群易受攻击，强烈建议尽快升级 containerd 版本。

# 对于Kubernetes集群升级 containerd 版本的内容

## Node节点排空 

```yaml
kubectl drain <node-to-drain> --ignore-daemonsets
```

## 停止Kubelet 和 Containerd 守护进程

```shell
systemctl stop kubelet
systemctl stop containerd
systemctl disable containerd.service --now
```

## 停止 contaired 关联进程

```shell
kill -9 `ps -aux | grep containerd | grep -v grep |awk '{print $2}'`
```

## 解压并替换Containerd二进制文件

```shell
tar -zxvf containerd-1.4.13-linux-amd64.tar.gz
cp bin/containerd /usr/bin/containerd
cp bin/containerd-shim /usr/bin/containerd-shim
cp bin/containerd-shim-runc-v1 /usr/bin/containerd-shim-runc-v1
cp bin/containerd-shim-runc-v2 /usr/bin/containerd-shim-runc-v2
cp bin/ctr /usr/bin/ctr
```

## 重启Containerd和Kubelet进程

```shell
systemctl start containerd
systemctl start kubelet
systemctl enable containerd
```

## 等待Pod重建及Node节点转为Ready状态

```shell
root@node1:~# kubectl get node -owide
NAME    STATUS   ROLES                  AGE    VERSION   INTERNAL-IP    EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION       CONTAINER-RUNTIME
node1   Ready    control-plane,master   344d   v1.20.6   192.168.2.10   <none>        Ubuntu 18.04.3 LTS   4.15.0-175-generic   containerd://1.4.13
node2   Ready    control-plane,master   344d   v1.20.6   192.168.2.11   <none>        Ubuntu 18.04.4 LTS   4.15.0-169-generic   containerd://1.4.13
node3   Ready    <none>                 344d   v1.20.6   192.168.2.12   <none>        Ubuntu 18.04.4 LTS   4.15.0-169-generic   containerd://1.4.13
node4   Ready    <none>                 344d   v1.20.6   192.168.2.13   <none>        Ubuntu 18.04.4 LTS   4.15.0-175-generic   containerd://1.4.13

```

