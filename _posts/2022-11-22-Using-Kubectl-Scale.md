---
layout:     post
title:     Kubectl Scale的最佳实践
subtitle:  Using Kubectl Scale Tutorial and Best Practices
date:       2022-11-22
author:     J
catalog: true
tags:
    - Kubernetes
---


kubectl `scale` 是帮助您管理 Kubernetes Deployment的众多工具之一。
在本文中，您将了解如何使用该工具，以及使用的最佳实践。

kubectl `scale`命令用于通过调整正在运行的pod（或者Container）的数量来立即扩展您的应用程序。这是增加Deployment副本数的最快、最简单的方法，它可用于应对需求高峰或长时间的静默。

在本文中，您将看到如何使用kubectl `scale`来扩展一个简单的Deployment。
您还将了解在需要更复杂变化时可以使用的选项。
最后，您将了解运行kubectl scale的最佳实践，以及调整 Kubernetes 副本`replicas`数量的一些替代方法。

## kubectl scale 使用示例
kubectl scale 命令用于更改 Kubernetes Deployment、Replicaset、Replication Controller和有Statefulset 中运行的副本数。当您增加副本数时，Kubernetes将启动新的pod以扩展您的服务。降低副本数将导致 Kubernetes 优雅地终止一些 pod，释放集群资源。

您可以运行kubectl scale来手动调整应用程序的副本数，以响应不断变化的服务容量需求。增加的流量负载可以通过增加副本数来处理，提供更多的应用程序实例来服务用户流量。当激增消退时，可以减少副本的数量。这有助于通过避免使用不需要的资源来降低成本。

## 使用 kubectl
kubectl `scale`的最基本用法如下所示：

```shell
$ kubectl scale --replicas=3 deployment/demo-deployment
```
执行此命令将调整名为demo-deployment 的Deployment，使其具有三个正在运行的副本。您可以通过替换不同资源类型和资源名称来执行相应的命令：

```
# ReplicaSet
$ kubectl scale --replicas=3 rs/demo-replicaset

# ReplicationController
$ kubectl scale --replicas=3 rc/demo-replicationcontroller

# StatefulSet
$ kubectl scale --replicas=3 sts/demo-statefulset
```

## 基本Scale缩放
现在我们将查看使用kubectl `scale`扩展部署的完整示例。这是一个定义简单Deploymenet的 `YAML` 文件：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: demo-app
  template:
    metadata:
      labels:
        app: demo-app
    spec:
      containers:
        - name: nginx
          image: nginx:latest
```
将此 YAML 保存到工作目录中的demo-deployment.yaml 。接下来，使用kubectl将部署添加到您的集群：

```shell
root@node1:~/demo-deployment# kubectl apply -f demo-deployment.yaml
deployment.apps/demo-deployment created

root@node1:~/demo-deployment# kubectl get pod
NAME                               READY   STATUS    RESTARTS   AGE
demo-deployment-6b756cbbc4-zw5fz   1/1     Running   0          3m50s
```
只有一个 pod 正在运行。这是符合预期的，因为Deployment的清单在其`spec.replicas`字段中声明了数量为`1`的副本。

单个副本不足以用于生产应用程序。如果pod 的所在Node节点出于任何原因宕机，您可能会遇到应用程序停止提供服务。使用kubectl scale增加副本数以提供更多空间：

```
root@node1:~/demo-deployment# kubectl scale --replicas=5 deployment/demo-deployment
deployment.apps/demo-deployment scaled

```
重复get pods命令以确认部署已成功扩展：

```shell
root@node1:~/demo-deployment# kubectl get pod
NAME                               READY   STATUS              RESTARTS   AGE
demo-deployment-6b756cbbc4-hkm7f   0/1     ContainerCreating   0          28s
demo-deployment-6b756cbbc4-r8dk6   0/1     ContainerCreating   0          28s
demo-deployment-6b756cbbc4-vn8tc   0/1     ContainerCreating   0          28s
demo-deployment-6b756cbbc4-zmvn2   0/1     ContainerCreating   0          28s
demo-deployment-6b756cbbc4-zw5fz   1/1     Running             0          10m

```

现在有五个 pod 正在运行用于demo-deployment部署。从`AGE`列可以看出， scale命令保留了原来的 pod，新增4个。

经过进一步考虑，您可能会决定此应用程序不需要5个副本。它只运行静态 NGINX Web 服务器，因此每个用户请求的资源消耗应该很低。再次使用scale命令来减少为3副本数并避免浪费集群容量：


```shell
root@node1:~/demo-deployment# kubectl scale --replicas=3 deployment/demo-deployment
deployment.apps/demo-deployment scaled
root@node1:~/demo-deployment# kubectl get pod
NAME                               READY   STATUS    RESTARTS   AGE
demo-deployment-6b756cbbc4-hkm7f   1/1     Running   0          109s
demo-deployment-6b756cbbc4-vn8tc   1/1     Running   0          109s
demo-deployment-6b756cbbc4-zw5fz   1/1     Running   0          12m
```
Kubernetes 已将两个正在运行的 pod 标记为终止。这会将正在运行的副本计数减少到请求的三个 pod。被选择驱逐的 pod 会被发送一个`SIGTERM`信号并被允许优雅地终止。一旦它们停止，它们将从 pod 列表中删除。

## 条件缩放
有时您可能想要扩展资源，但前提是已经有特定数量的副本在运行。这可以避免无意中覆盖之前的缩放更改，例如集群中其他用户所做的更改。

在命令中包含`--current-replicas`标志以使用此行为：

```shell
root@node1:~/demo-deployment# kubectl scale --current-replicas=3 --replicas=5 deployment/demo-deployment
deployment.apps/demo-deployment scaled
root@node1:~/demo-deployment# kubectl get pod
NAME                               READY   STATUS              RESTARTS   AGE
demo-deployment-6b756cbbc4-hkm7f   1/1     Running             0          9m36s
demo-deployment-6b756cbbc4-jx52p   0/1     ContainerCreating   0          2s
demo-deployment-6b756cbbc4-vn8tc   1/1     Running             0          9m36s
demo-deployment-6b756cbbc4-x2qjc   0/1     ContainerCreating   0          2s
demo-deployment-6b756cbbc4-zw5fz   1/1     Running             0          20m

```

此示例将演示部署部署扩展到五个副本，但前提是当前有三个副本正在运行。`--current -replicas`需要总是精确匹配；您不能将条件表示为“小于”或“大于”特定计数。

## 扩展多个资源

当您提供多个名称作为参数时， kubectl `scale`命令可以一次扩展多个资源。每个资源都将缩放到由`--replicas`标志设置的相同副本数。

```shell
$ kubectl scale --replicas=5 deployment/app deployment/database
deployment.apps/app scaled
deployment.apps/database scaled
```
此命令将应用程序和数据库部署分别扩展到五个副本。

另外您可以通过提供`--all`标志来扩展特定类型的每个资源，例如此示例以扩展您的默认命名空间`default`中的所有Deployment：

```shell
$ kubectl scale --all --replicas=5 --namespace=default deployment
deployment.apps/app scaled
deployment.apps/database scaled
```
这会选择当前活动命名空间内的每个匹配资源。缩放的对象显示在命令的输出中。
您可以获得对使用`--selector`标志缩放的对象的精细控制。这使您可以使用标准选择语法根据对象的标签过滤对象。下面是一个使用`app-name=demo-app`签扩展所有部署的示例：


```shell
$ kubectl scale --replicas=5 --selector=app-name=demo-app deployment
deployment.apps/app scaled
deployment.apps/database scaled

```

## 更改超时 `--timeout`
`--timeout`标志设置 Kubectl 在放弃缩放操作之前将等待的时间。默认情况下，没有等待期。该标志接受人类可读格式的时间值，例如`5m`或`1h`：

```shell
kubectl scale --replicas=5 --timeout=1m deployment/demo-deployment
```
如果无法立即完成缩放更改，这可以让您避免长时间的终端挂起。尽管kubectl scale是命令式命令，但在将新 pod 调度到节点时，对缩放的更改有时可能需要几分钟才能完成。


## 最佳实践
使用kubectl scale通常是扩展工作负载的最快、最可靠的方法。但是，为了安全操作，需要记住一些最佳实践。
这里有一些提示。

- 避免过于频繁地缩放。
副本计数的更改应响应特定事件，例如导致请求运行缓慢或被丢弃的拥塞。最好分析您当前的服务容量，估计满意地处理所有流量所需的容量，然后在顶部添加额外的缓冲区以预测任何未来的增长。避免过于频繁地扩展您的应用程序，因为每个操作都可能导致 Pod 被调度和终止时的延迟。
- 缩小到零将停止您的应用程序。
您可以运行kubectl scale `--replicas=0`，这将删除所选对象中的所有容器。您可以通过使用正值重复命令来再次向上扩展。
- 确保您选择了正确的对象。
没有确认提示，因此请务必注意您选择的对象。按名称手动选择对象是最安全的方法，可以防止您意外缩放应用程序的其他部分，这可能会导致中断或浪费资源。
- 使用 `--current-replicas`来避免意外。
使用--current-replicas标志通过确保仅当当前计数符合您的预期时才更改比例来提高安全性。否则，您可能会无意中覆盖其他用户或 Kubernetes autoscaler 应用的缩放更改。


## kubectl Scale的替代品
运行kubectl `scale`是对集群有直接影响的命令式操作。您指示 Kubernetes 尽快提供特定数量的副本。如果您使用命令式kubectl create命令创建对象，这是合乎逻辑的，但如果您最初使用声明性 YAML 文件运行kubectl apply ，则这是不合适的，如上所示。运行scale命令后，集群中的副本数量将与 YAML 的`spec.replicas`字段中定义的数量不同。更好的做法是修改YAML 文件，然后将其重新应用到您的集群。

首先将`spec.replicas`字段更改为新的所需副本数：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-deployment
spec:
  replicas: 5   # edit
  selector:
    matchLabels:
      app: demo-app
  template:
    metadata:
      labels:
        app: demo-app
    spec:
      containers:
        - name: nginx
          image: nginx:latest
```
现在对修改后的文件重复kubectl apply命令：

```shell
$ kubectl apply -f demo-deployment.yaml
```
kubectl 将自动比较更改并采取措施使集群的状态朝着文件中声明的方向发展。这将导致 pod 被自动创建或终止，因此运行实例的数量再次与spec.replicas字段匹配。

kubectl scale的另一个替代方案是 Kubernetes 对`hpa`的支持。配置此机制允许 Kubernetes 根据 CPU 使用率和网络活动等指标在配置的最小值和最大值之间自动调整副本计数。

## 最后的想法

kubectl scale命令是一种用于扩展 Kubernetes Deployment、ReplicaSet、Replication Controller和StatefulSet的命令式机制。它在每次调用时以一个或多个对象为目标并缩放它们，以便运行指定数量的 pod。您可以选择设置一个条件，以便只有在存在特定数量的现有副本时才会更改比例，从而避免在错误的方向上无意中调整大小。
