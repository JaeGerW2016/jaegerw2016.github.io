---
title: Kubernetes的Pod调度算法
date: 2019-02-28 02:20:46
tags: kubernetes
---
### Kubernetes的Pod调度算法

在本文档中，解释了如何为Pod选择节点的算法。在选择Pod的目标节点之前有两个步骤。第一步是过滤所有节点（预选），第二步是对剩余节点进行排名（优选），以找到最适合Pod的节点。

### 预选 
预选的目的是过滤掉不满足Pod某些要求的节点。例如，如果节点上的空闲资源（通过容量减去已经在节点上运行的所有Pod的资源请求的总和来测量）小于Pod的所需资源，则不应在优选阶段考虑该节点打分排名，就被过滤掉了。目前，有几个“预选策略”实现了不同的过滤策略
-  `NoDiskConflict` 检查pod请求的卷以及已挂载的卷存在冲突。如果这个主机已经挂载了卷，其它同样使用这个卷的Pod不能调度到这个主机上。目前支持的卷包括：AWS EBS，GCE PD，ISCSI和Ceph RBD。**仅检查这些受支持类型的持久卷声明。直接添加到pod的持久卷不会被评估，也不受此策略的约束。**
-  `NoVolumeZoneConflict`  检查给定的zone限制前提下，检查如果在此主机上部署Pod是否存在卷冲突。
-  `PodFitsResources`  检查可用资源（CPU和内存）是否满足Pod的要求。测量空闲资源是通过节点容量减去节点上所有Pod的请求总和。要了解有关Kubernetes中资源QoS的更多信息，请查看[QoS提议](https://github.com/kubernetes/community/blob/master/contributors/devel/design-proposals/node/resource-qos.md)。
-  `PodFitsHostPorts`  检查Pod所需的任何HostPort是否已在节点上占用
-  `HostName`  筛选出除`PodSpec`的``NodeName`字段中指定的节点之外的所有节点 简单来说就是检查主机名称是不是Pod指定的`HostName`
-  `MatchNodeSelector` 检查节点的标签是否与Pod的`nodeSelector`字段中指定的标签匹配，并且从Kubernetes v1.2开始，也匹配`nodeAffinity`if present。有关两者的详细信息，请参见[此处](https://kubernetes.io/docs/user-guide/node-selection/)。
-  `MaxEBSVolumeCount`  确保挂载的ElasticBlockStore卷的数量不超过最大值（默认情况下为39，因为Amazon建议最多40个，其中40个为根卷保留其中一个 - 请参阅[Amazon的文档](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/volume_limits.html#linux-specific-volume-limits)）。可以通过设置`KUBE_MAX_PD_VOLS`环境变量来控制最大值。
-  `MaxGCEPDVolumeCount`  确保挂载的GCE PersistentDisk卷的数量不超过最大值（默认情况下，16，这是GCE允许的最大值 - 请参阅[GCE的文档](https://cloud.google.com/compute/docs/disks/persistent-disks#limits_for_predefined_machine_types)）。可以通过设置`KUBE_MAX_PD_VOLS`环境变量来控制最大值。
-  `CheckNodeMemoryPressure`  检查是否可以在提示`memory pressure`的节点上安排pod。目前，`BestEffort`这种Qos等级的pod不能被调度到 `memory pressure`的节点上，因为它会被kubelet自动驱逐。
-  `CheckNodeDiskPressure`  检查是否可以在提示`disk pressure`节点上安排Pod。目前，Pod不会被调度在disk pressure`节点上，因为它会被kubelet自动驱逐。

上述预选的详细信息可以在[pkg / scheduler / algorithm / predicates / predicates.go中找到](http://releases.k8s.io/HEAD/pkg/scheduler/algorithm/predicates/predicates.go)。上面提到的所有策略可以组合使用以执行复杂的过滤策略。默认情况下，Kubernetes使用这些谓词中的一些，但不是全部。您可以在[pkg / scheduler / algorithmprovider / defaults / defaults.go中](http://releases.k8s.io/HEAD/pkg/scheduler/algorithmprovider/defaults/defaults.go)查看[默认使用的](http://releases.k8s.io/HEAD/pkg/scheduler/algorithmprovider/defaults/defaults.go)。

### 优选

预选节点被认为适合托管Pod，并且通常存在多个节点。Kubernetes优先考虑这些节点，以找到Pod的“最佳”节点。优先级由一组优先级函数执行。对于每一个节点，优先级函数给出从0-10开始的分数，其中10代表“最优选”，0代表“最不优选”。每个优先级函数由正数加权，每个节点的最终得分通过将所有加权得分相加来计算。例如，假设有两个优先级的功能，`priorityFunc1`和`priorityFunc2`与加权系数`weight1`和`weight2`分别一些NodeA上的最后得分是：

```
finalScoreNodeA = (weight1 * priorityFunc1) + (weight2 * priorityFunc2)
```

在计算所有节点的分数之后，选择具有最高分数的节点作为Pod的主机。如果存在多个具有相同最高分数的节点，则选择其中的随机节点。

目前，Kubernetes调度程序提供了一些实用的优先级功能，包括：

- `LeastRequestedPriority`：如果新的pod要分配给一个节点，这个节点的优先级就由节点空闲的那部分与总容量的比值（即（总容量-节点上pod的容量总和-新pod的容量）/总容量）来决定。CPU和内存的权重相等，比值最大的节点的得分最高。需要注意的是，这个优先级函数起到了按照资源消耗来跨节点分配pods的作用。计算公式如下：
```
score = cpu((capacity – sum(requested)) * 10 / capacity) + memory((capacity – sum(requested)) * 10 / capacity) / 2
```

- `BalancedResourceAllocation`：尽量选择在部署Pod后各项资源更均衡的机器，计算公式如下：
```
score = 10 – abs(cpuFraction-memoryFraction)*10
```
- `SelectorSpreadPriority`：对于属于同一个service、replication controller的Pod，尽量分散在不同的主机上。如果指定了区域，则会尽量把Pod分散在不同区域的不同主机上
- `CalculateAntiAffinityPriority`：对于属于同一个service的Pod，尽量分散在不同的具有指定标签的主机上
- `ImageLocalityPriority`：根据主机上是否已具备Pod运行的环境来打分。`ImageLocalityPriority`会判断主机上是否已存在Pod运行所需的镜像，根据已有镜像的大小返回一个0-10的打分。如果主机上不存在Pod所需的镜像，返回0；如果主机上存在部分所需镜像，则根据这些镜像的大小来决定分值，镜像越大，打分就越高。
- `NodeAffinityPriority`：（Kubernetes v1.2）Kubernetes调度中的亲和性机制。Node Selectors（调度时将pod限定在指定节点上），支持多种操作符（In, NotIn, Exists, DoesNotExist, Gt, Lt），而不限于对节点labels的精确匹配。
  另外，Kubernetes支持两种类型的选择器：
  一种是“hard（requiredDuringSchedulingIgnoredDuringExecution）”选择器，它保证所选的主机必须满足所有Pod对主机的规则要求。这种选择器更像是之前的nodeselector，在nodeselector的基础上增加了更合适的表现语法。
  另一种是“soft（preferresDuringSchedulingIgnoredDuringExecution）”选择器，它作为对调度器的提示，调度器会尽量但不保证满足NodeSelector的所有要求。
  Kubernetes调度中的亲和性机制请看[这里](https://www.qikqiak.com/post/kubernetes-affinity-scheduler/)了解更多详情。

#### 参考文档：

https://github.com/kubernetes/community/blob/master/contributors/devel/sig-scheduling/scheduler_algorithm.md

https://blog.csdn.net/WaltonWang/article/details/54604392
