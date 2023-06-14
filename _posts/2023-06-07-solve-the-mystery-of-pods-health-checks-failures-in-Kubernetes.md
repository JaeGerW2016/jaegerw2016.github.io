---
layout: mypost
title: Kubernetes中Pod health checks失败排查
categories: [Kubernetes,Network]
---

## 背景
Kuberntes中会给Pod或者Deployment的加上health check健康检测，用于提供高可用的服务，最近注意到 pod 健康检查有时会在没有任何明显报错的原因的情况下失败，，由于pod会在这种情况下立即重建，然后几乎立即恢复。没有人认为这是一个问题，因为它很少发生并且不会影响任何事情。

最近观察到一个集群里的pod频繁重启，已经影响到服务的可用性，所以就对这个问题开始重视起来了。

## 检查日志
- 节点Node的日志、Event信息
- kubelet 日志
- 登陆最近健康检查失败的Pod的日志
- 登陆其他关联服务依赖的Pod的日志
- Containerd 日志

> kubectl logs -n namespace --previous xxx #pod name 可以查看CrashLookBackOff的Pod日志

基于以上所有的日志，都查了一遍，没有明显的日志提示报错信息，但是Pod因健康检查失败而重启的事件还在时常发生，而且现象随着Pod数量运行数量增多而变得更频繁。


## 深入了解
更大的可能性来自于网络问题，而网络问题也是Kubernetes中最复杂的，需要层层抽丝剥茧的排查，涉及网络的排查，就得求助于tcpdump抓包

## tcpdump抓包
在抓Pod流量包的过程中，根据TCP的三次握手原则，TCP 是面向连接的协议，所以使用 TCP 前必须先建立连接，而建立连接是通过三次握手来进行的。注意到当从 Kubelet 到 pod 的 TCP SYN 时，pod 回复了 TCP SYN-ACK，但从Kubelet 发出TCP ACK 并没有回应。然后再经过几次重试后，Kubelet 才建立了一个没有问题的 TCP 会话。这个现象存在一定的随机性。

## SS 服务端口检测
我们根据在Node节点上`ss -natp`的输出。这应该已经显示每个进程的连接数。

我们很快发现失败的连接被卡在 `SYN-SENT`状态中。这与我们在 tcpdump 中看到的不匹配，它应该是`SYN-RECV`状态。

## 检查Conntrack表
Linux中的连接跟踪模块`Conntrack`

用于维护可跟踪协议（trackable protocols）的连接状态。 也就是说，连接跟踪针对的是特定协议的包，而不是所有协议的包。

在 Conntrack 中，断开的连接卡在 SYN-RECV 中，这至少是意料之中的。

可以想象是什么阻止 TCP SYN-ACK 返回到达 Kubelet 打开的套接字？

通过在Node节点上的CLI命令行执行`conntrack -L -o ktimestamp`

可以看到`SYN-SENT`状态或`SYN-RECV`状态中的连接并不是完全随机的，因为所有源端口`30xxx`或者`31xxx` 看起来都很熟悉。

此时我们检查Linux中的sysctl参数`ip_local_port_range`
```
# cat /proc/sys/net/ipv4/ip_local_port_range
12000 65001
```


## ipvs检查service后端服务端口
我们用 ipvsadm 检查了我们的 ipvs 配置，发现卡住连接中的所有端口，类似`31xxx`和`30xxx`都被 Kubernetes nodeports 服务保留，也就是kubelet发起的连接的端口和Nodeport服务端口二者起冲突。

## 根本原因
Kubelet 使用随机源端口（12000-65001段）向 pod 发起 TCP 会话。例如，31055。TCP SYN 到达 pod，并且 pod 使用 TCP SYN-ACK 回复到端口 31055。

回复命中 IPVS，我们在其中使用Nodeport 31055 为 Kubernetes 服务提供负载均衡器。TCP SYN-ACK 被重定向到服务端点（其他 pod）。结果是可以预见的：tcp三次握手失败，tcp会话无法建立。


## 解决方案
健康检查失败的解决方案是**禁止使用Nodeport端口范围作为 TCP 会话的源端口**

```
echo "30000-32767" > /proc/sys/net/ipv4/ip_local_reserved_ports
```
将`30000-32767`段端口预留做为Nodeport的服务端口，避免端口冲突。

**最终检查配置项**：

```
root@node1:~# cat /proc/sys/net/ipv4/ip_local_port_range
32768   60999
root@node1:~# cat /proc/sys/net/ipv4/ip_local_reserved_ports
30000-32767
```
