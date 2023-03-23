---
layout:     post
title:     在Kubernetes中实现零停机部署
subtitle:  Achieving Zero-Downtime deployment in Kubernetes
date:       2022-09-16
author:     J
catalog: true
tags:
    - Kubernetes
---
多年来，Kubernetes 已成为容器编排的首选平台。默认情况下，Kubernetes 使用滚动更新策略（rolling update ）进行部署。此策略旨在防止停机，确保某些容器实例在执行更新时随时启动并运行。只有在新版本的容器准备好接收实时流量后，旧版本的容器才会关闭。

虽然理论上这似乎是正确的，但有一个转折注意点。让我们详细了解一下。

Kubernetes Pod 驱逐流程
在开始 pod eviction 过程之前，让我们了解 Kubernetes 的两个主要组件，它们在 eviction 过程中起着非常重要的作用：

Kubelet — Kubelet 收集 pod 的所有详细信息，例如 IP 地址，并将它们报告回`control-plane`，并不断轮询`Watch` `control-plane` 以获取更新。

Endpoint — Kube-proxy 使用Endpoint在节点上设置 iptables 规则。每次Endpoint发生变化时，kube-proxy 都会检索新的 IP 地址、端口列表并配置新的 iptables 规则。

![Application Downtime](https://miro.medium.com/max/700/0*2FnRX1PqBYBuXE8o.png)

当启动 pod eviction 过程时，`API Server`将etcd数据库中的 pod 状态修改为Terminating状态。节点`Node`的kubelet和endpoints-controller持续监控 pod 的状态。一旦他们注意到终止状态，他们就会无序启动驱逐过程（以下两个操作都是异步的）：

- Kubelet 发送SIGTERM信号终止 pod
- Endpoints-controller 处理Endpoint删除过程，摘除被删除的endpoint以停止传入流量

以上2个操作由于是无序的，Endpoints-controller启动诺干个goroutine执行从service queue中pop service进行syncService同步，完成一次sync后等待1s再从service queue中pop一个service进行sync，如此反复。

现在是转折注意点。

如果Endpoint删除过程及Iptable规则更新在Pod的 SIGTERM 信号之前完成，则在容器终止时不会有新请求到达（期望达到的预想）。

但是，如果容器在Endpoint删除过程之前开始终止，容器将继续接收Iptabels转发流量（直到syncService同步完成，Endopoint被删除，Iptables规则不转发到被删除的Endpoint），导致应用程序停机，因为 Kubernetes 仍在将流量路由到 IP 地址，但 pod 不再存在。

Graceful shutdown（优雅关闭）
我们需要通过关闭所有持久连接（数据库、队列、Websocket 等）来确保 Pod 正常终止，并等待所有活动请求耗尽。

解决方案——pre-stop

为了实现优雅关闭，我们需要确保我们能够优雅地处理SIGTERM（终止 pod 的信号） 和SIGKILL（强制终止 pod 的信号） 命令。

我们可以通过以下两种方式实现 pre-stop hooks 来实现这一点：

- 在 pre-stop 钩子中添加`sleep`睡眠时间， pod 驱逐过程并等待Endpoint移除过程，这将延迟SIGTERM信号并为Endpoint移除传播创造时间。对于大多数情况，5-10 秒之间的值就足够了。
- 设置 `terminateGracePeriodSeconds` — 这是 kubelet 在强制终止容器之前等待的时间限制。根据您的应用程序和集群情况，这建议配置在 15 到 45 秒之间变化。

![Zero Downtime](https://miro.medium.com/max/700/0*BFAc-ZbcHf-jEVQC.png)
使用上述两个选项，我们可以确保在 Pod 终止之前优雅地处理所有正在进行的请求，从而实现零停机时间。
