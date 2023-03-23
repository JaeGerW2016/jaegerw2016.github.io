---
layout:     post
title:      Kubernetes Upgrade不得不知道的API资源变化
subtitle:   Kubernetes升级：如何处理api变化 
date:       2020-04-27
author:     J
catalog: true
tags:
    - kubernetes
---

随着Kubernetes 版本不断迭代发布，API会定期进行重组或升级。随着API的发展，旧的API会被弃用并最终被删除。

*V1.16*版本将有以下的API被废弃和支持新的更稳定的API版本

#### NetworkPolicy 
~~不再使用**extensions / v1beta1** API~~

- 迁移到从V1.8+可用的**networking.k8s.io/v1** API

#### PodSecurityPolicy

从 **extensions/v1beta1** API 迁移到从V1.10+可用的**policy/v1beta1** API

#### DaemonSet 

~~不再使用**extensions/v1beta1** and **apps/v1beta2** API~~

- 迁移到从V1.9+可用的 **apps/v1** API

其他字段的变化

- `spec.templateGeneration` 已移除
- `spec.selector`现在是必需的，创建后是不变的；使用现有模板标签作为无缝升级的选择器
- `spec.updateStrategy.type`现在默认为`RollingUpdate`（默认`extensions/v1beta1`为`OnDelete`）

#### Deployment

~~不再使用**extensions/v1beta1**, **apps/v1beta1**, and **apps/v1beta2** API~~

- 迁移到V1.9+可用的 **apps/v1** API

其他字段变化

- `spec.rollbackTo` 已移除
- `spec.selector`现在是必需的，创建后是不变的；使用现有模板标签作为无缝升级的选择器
- `spec.progressDeadlineSeconds`现在默认为`600`秒（默认`extensions/v1beta1`为无截止日期）
- `spec.revisionHistoryLimit`现在默认为`10`（默认`apps/v1beta1`为`2`，默认`extensions/v1beta1`为保留所有内容）
- `maxSurge`而`maxUnavailable`现在默认为`25%`（默认`extensions/v1beta1`为`1`）

#### StatefulSet

~~不再使用**extensions/v1beta1** and **apps/v1beta2** API~~

- 迁移到从V1.9+可用的 **apps/v1** API

其他字段变化

- `spec.selector`现在是必需的，创建后是不变的；使用现有模板标签作为无缝升级的选择器
- `spec.updateStrategy.type`现在默认为`RollingUpdate`（默认`apps/v1beta1`为`OnDelete`）

#### ReplicaSet

~~不再使用**extensions/v1beta1**, **apps/v1beta1**, and **apps/v1beta2** API~~

- 迁移到从V1.9+可用的 **apps/v1** API

其他字段变化

`spec.selector`现在是必需的，创建后是不变的；使用现有模板标签作为无缝升级的选择器

> 在**V1.22**版本 **extensions / v1beta1** API 将被废弃，迁移至v1.14+可用的**networking.k8s.io/v1beta1** API



### 如何转换API

可以使用以下 `kubectl convert`命令自动转换现有对象： `kubectl convert -f <file> --output-version <group>/<version>`

例如，要将旧的Deployment转换为apps / v1，可以运行： `kubectl convert -f ./my-deployment.yaml --output-version apps/v1` 请注意，这可能使用非理想的默认值。要了解有关特定资源的更多信息，请查看Kubernetes [api参考](https://kubernetes.io/docs/reference/#api-reference)

可以通过在禁用上述资源的情况下启动apiserver来测试集群，以模拟即将进行的删除。将以下标志添加到apiserver启动参数中：

```
--runtime-config=apps/v1beta1=false,apps/v1beta2=false,extensions/v1beta1/daemonsets=false,extensions/v1beta1/deployments=false,extensions/v1beta1/replicasets=false,extensions/v1beta1/networkpolicies=false,extensions/v1beta1/podsecuritypolicies=false
```