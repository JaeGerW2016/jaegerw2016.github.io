---
layout:     post
title:      Service Topology 服务拓扑
subtitle:   了解Service流量走向 
date:       2020-04-01
author:     J
catalog: true
tags:
    - kubernetes
---

# Service Topology 服务拓扑

###### 功能状态： `Kubernetes v1.17` [α](https://kubernetes.io/docs/concepts/services-networking/service-topology/#)

service topology 服务拓扑能使kubernetes的service基于集群的node拓扑来路由流量。例如service可以指定将流量优先路由到与client客户端同一节点或者在同一可用区域的node上的pod。

### 先决条件

为了启用拓扑感知的服务路由，需要满足以下先决条件：

- Kubernetes 1.17或更高版本

- [Kube-proxy](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-proxy/)在iptables模式或IPVS模式下运行

- 启用[EndpointSlices](https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/)

- 为所有Kubernetes组件启用`ServiceTopology`和`EndpointSlice`功能门：

  ```
  --feature-gates="ServiceTopology=true,EndpointSlice=true"
  ```

## 介绍

默认情况下，发送到`ClusterIP`或`NodePort`服务的流量通过`iptables`或者`ipvs`转发规则随机路由到该服务的任何后端endpoints地址。不支持更加复杂灵活的拓扑（比分：分区路由）

 *Service Topology*功能允许服务建立者定义路由基于`Node labels`对源和目的节点流量做有针对的策略路由规则。

通过使用源和目的地之间的`Node labels`匹配，操作员可以针对一些特定metrics来指定物理拓扑（机架位置）或者网络拓扑（可用区域）上的“更近”和“更远”的一组节点。

例如，对于公共云中的许多运营商而言，倾向于将服务流量保持在同一区域内，因为区域间流量具有与其相关的成本，而区域内流量则没有。其他常见需求包括能够将流量路由到由DaemonSet管理的本地Pod，或将流量保持到连接到同一机架顶部交换机的节点，以实现最低延迟。

## 使用服务拓扑

如果您的集群启用了服务拓扑，则可以通过`topologyKeys`在服务规范上指定字段来控制服务流量路由。该字段是节点标签的优先顺序列表，将在访问此服务时用于对端点进行排序。流量将被定向到其第一个标签的值与该标签的始发节点的值匹配的节点。如果在匹配的节点上没有该服务的后端，则将考虑第二个标签，依此类推，直到没有标签剩余为止。

如果找不到匹配项，则流量将被拒绝，就像该服务根本没有后端一样。即，基于具有可用后端的第一个拓扑密钥选择端点。如果指定了此字段，并且所有条目都没有与客户端拓扑匹配的后端，则该服务没有该客户端的后端，因此连接将失败。该特殊值`"*"`可以用来表示“任何拓扑”。如果使用了此包罗万象的值，则仅作为列表中的最后一个值才有意义。

如果`topologyKeys`未指定或为空，则不会应用拓扑约束。

考虑一个带有节点的群集，这些节点用其主机名，区域名称和区域名称标记。然后，您可以如下设置`topologyKeys`服务的值以引导流量。

- 只有到了同一节点上的端点，如果节点上不存在任何端点失败： `["kubernetes.io/hostname"]`。
- 优先于同一节点上的端点，回落至终点在同一个区域，随后在相同区域，否则失败：`["kubernetes.io/hostname", "topology.kubernetes.io/zone", "topology.kubernetes.io/region"]`。例如，在数据局部性很关键的情况下，这可能很有用。
- 优先使用同一区域，但如果此区域内没有可用端点，则回退到任何可用端点上 `["topology.kubernetes.io/zone", "*"]`。

## 约束条件

- 服务拓扑与不兼容`externalTrafficPolicy=Local`，因此服务不能同时使用这两个功能。可以在不同服务上的同一群集中使用这两个功能，而不能在同一服务上使用。
- 当前有效的拓扑密钥仅限于`kubernetes.io/hostname`， `topology.kubernetes.io/zone`和`topology.kubernetes.io/region`，但将来会推广到其他节点标签。
- 拓扑密钥必须是有效的标签密钥，并且最多可以指定16个密钥。
- 如果使用了全部捕获值`"*"`，则它必须是拓扑键中的最后一个值。

以下是使用服务拓扑功能的常见示例。

### 仅节点本地端点

仅路由到节点本地终结点的服务。如果节点上不存在端点，则会丢弃流量：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
  topologyKeys:
    - "kubernetes.io/hostname"
```

### 首选节点本地端点

首选节点本地终结点但如果节点本地终结点不存在则回退到群集范围终结点的服务：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
  topologyKeys:
    - "kubernetes.io/hostname"
    - "*"
```

### 仅区域或区域端点

优先选择区域端点而不是区域端点的服务。如果两个端点均不存在端点，则会丢弃流量。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
  topologyKeys:
    - "topology.kubernetes.io/zone"
    - "topology.kubernetes.io/region"
```

### 优先选择本地节点，区域节点，然后是区域端点

一种服务，它偏爱节点本地，区域终结点和区域终结点，但回退到群集范围的终结点。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
  topologyKeys:
    - "kubernetes.io/hostname"
    - "topology.kubernetes.io/zone"
    - "topology.kubernetes.io/region"
    - "*"
```