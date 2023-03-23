---
title: 为您的应用程序指定PodDisruptionBudget
date: 2019-11-06 16:53
tag: kubernetes
---

## 背景概念

在Kubernetes中，为了保证业务不中断或业务SLA不降级，需要将应用进行集群化部署。通过PodDisruptionBudget控制器可以设置应用POD集群处于运行状态最低个数，也可以设置应用POD集群处于运行状态的最低百分比，这样可以保证在主动销毁应用POD的时候，不会一次性销毁太多的应用[POD](https://www.kubernetes.org.cn/kubernetes-pod)，从而保证业务不中断或业务SLA不降级。

PodDisruptionBudget是kubernetes 1.7版本之后的新特性

支持kubernetes内置控制器类型：

- deployment

- ReplicationController

- ReplicaSet

- StatefulSet

  > 1.15版本之后 支持custom controller，对于static pod或者裸Pod 不受以上控制器控制，在定义PDB的有一定的区别

## 指定PodDisruptionBudget

`PodDisruptionBudget`具有三个字段：

- `.spec.selector`标签选择器用于指定要应用的Pod集合。这是**必填字段**。

- `.spec.minAvailable`这是对从该集合中移出的豆荚数量的描述，即使在没有逐出的豆荚的情况下，该集合在驱逐后仍必须可用。`minAvailable`可以是绝对数字或百分比。

- `.spec.maxUnavailable`（在Kubernetes 1.7和更高版本中可用），它描述了该集合在驱逐后可能不可用的Pod数量。它可以是绝对数字或百分比。

  > **注意：**对于1.8和更早版本：`PodDisruptionBudget` 使用`kubectl`命令行工具创建对象时，`minAvailable`如果`minAvailable`未`maxUnavailable`指定或未指定，则该字段的默认值为1 。

这里需要注意的是，`.spec.minAvailable`参数和`.spec.maxUnavailable`参数是互斥的，也就是说如果使用了其中一个参数，那么就不能使用另外一个参数了。

比如当进行kubectl drain或者POD主动逃离的时候，kubernetes可以通过下面几种情况来判断是否允许：

- minAvailable设置成了数值5：应用POD集群中最少要有5个健康可用的POD，那么就可以进行操作。

- minAvailable设置成了百分数30%：应用POD集群中最少要有30%的健康可用POD，那么就可以进行操作。

- maxUnavailable设置成了数值5：应用POD集群中最多只能有5个不可用POD，才能进行操作。

- maxUnavailable设置成了百分数30%：应用POD集群中最多只能有30%个不可用POD，才能进行操作。

在极端的情况下，比如将maxUnavailable设置成0，或者设置成100%，那么就表示不能进行kubectl drain操作。同理将minAvailable设置成100%，或者设置成应用POD集群最大副本数，也表示不能进行kubectl drain操作。

这里面需要注意的是，使用PodDisruptionBudget控制器并不能保证任何情况下都对业务POD集群进行约束，PodDisruptionBudget控制器只能保证POD主动逃离的情况下业务不中断或者业务SLA不降级，例如在执行`kubectl drain`命令时。

使用minAvailable的示例PDB：

```yaml
apiVersion: policy/v1beta1
kind: PodDisruptionBudget
metadata:
  name: zk-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: zookeeper
```

使用maxUnavailable（Kubernetes 1.7或更高版本）的示例PDB：

```yaml
apiVersion: policy/v1beta1
kind: PodDisruptionBudget
metadata:
  name: zk-pdb
spec:
  maxUnavailable: 1
  selector:
    matchLabels:
      app: zookeeper
```

## 任意控制器和选择器

如果仅将PDB与内置应用程序控制器（Deployment，ReplicationController，ReplicaSet和StatefulSet）一起使用，并且PDB选择器与控制器的选择器匹配，则可以跳过本节。

您可以将PDB与受另一种类型的控制器，“操作员”控制的Pod或裸Pod一起使用，但具有以下限制：

- 只能`.spec.minAvailable`使用，不能使用`.spec.maxUnavailable`。
- 只能使用整数值`.spec.minAvailable`，不能使用百分比。

您可以使用选择器来选择属于内置控制器的Pod的子集或超集。但是，当命名空间中有多个PDB时，必须注意不要创建选择器重叠的PDB。

## 参考文档

https://kubernetes.io/docs/tasks/run-application/configure-pdb/

https://www.kubernetes.org.cn/2486.html

https://kubernetes.io/docs/concepts/workloads/pods/disruptions/