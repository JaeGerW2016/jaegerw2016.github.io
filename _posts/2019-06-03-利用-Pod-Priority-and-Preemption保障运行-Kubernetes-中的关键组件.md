---
title: 利用 Pod Priority and Preemption保障运行 Kubernetes 中的关键组件
date: 2019-06-03 23:56:50
tags: kubernetes
---
Kubernets集群内部运行着众多各式各样不同的组件，某些组件会相比其他组件来说更为重要，缺少了这些组件，集群的核心功能或者用户业务将无法得到保障：比如 DNS 组件，当 DNS 组件运行异常，集群内部的 DNS 服务将不可用；又比如CNI网络组件，当网络组件异常，某个节点甚至集群的网络将不可用。

**Kubernetes 提供了不少机制来保证组件的正常运行，比如：**

- 通过配置 pod 的 `restartPolicy`，我们可以保证这个 pod 异常退出后重启策略，能保证重新运行起来；
- 通过配置 deployment 的 `replicas`，我们可以保证这个 pod 有指定个存活的多副本，防止单点故障；
- 通过配置 pod 的 `iveness`和 `readiness` 检测，我们可以及时发现 pod 的异常状态；
- 通过配置 pod 资源的 `requests` 和 `limits`，我们可以保证这个 pod 的Qos服务等级以确保有合适的资源来运行。
- 通过配置pod的标识为`Critical Pod`,标记关键组件,在某些原因被驱逐，处于等待调度时，配置 Critical pods 标记的组件可以确保被重新调度。

以上的几种机制下，基本保障组件的正常运行。

下面我们设定一种特殊情况：集群内的一个节点出现`NotReady`异常，控制器检测到这个节点异常后，开始重新调度异常节点上的 pod；假设这个时候，集群的其它节点可用的资源已经不多了，异常节点上有 10 个 pod 等待重新调度，调度完一半后，其它 3个 pod 因为没有可用资源无法调度，而在这 3 个等待调度的 pod 里，恰好有一个关键组件。这个时候，集群的运行因为核心应用缺失导致集群完整性功能得不到保障。

**[Guaranteed Scheduling For Critical Add-On Pods](https://kubernetes.io/docs/tasks/administer-cluster/guaranteed-scheduling-critical-addon-pods/#marking-add-on-as-critical)**

除了Kubernetes核心组件（如Master上运行的api-server，scheduler，controller-manager）之外，还有许多附加组件，由于各种原因，这些Add-On组件必须在集群Node节点（而不是Kubernetes主节点）上运行。其中一些附加组件对于功能完备的群集至关重要，例如Metrics-Server，DNS和UI。如果关键组件Pod被驱逐（手动或作为升级等其他操作的导致）并且导致集群完整性功能得不到保障（例如，当Kubernets集群处于高负载运行状态并且有其他待处理的Pod处于Pending状态），集群可能会停止正常工作并驱逐的关键组件Pod以腾出节点上可用的资源或者由于某些其他原因提供一定数量的节点资源

**标记Pod成为Critical Pod**

**条件1**

- 运行在`kube-system`命名空间
- Enable Feature Gate `ExperimentalCriticalPodAnnotation` `PodPriority`

- 必须加上Annotation `scheduler.alpha.kubernetes.io/critical-pod=""`（从kubernetes的1.13版本起，这个注释已被弃用，将在1.14版本中删除。）

**条件2**

- Enable Feature Gate `ExperimentalCriticalPodAnnotation` `PodPriority`

- Pod的Priority不为空，并将Pod的priorityClassName设置为`system-cluster-critical`或`system-node-critical`

  > 以上条件1和2满足其中一个即可

**[Pod Priority and Preemption](https://kubernetes.io/docs/concepts/configuration/pod-priority-preemption/)**

Priority 表明了 pod 相对于其它 pod 的重要程度，开启 pod priority 特性后，Kubernetes 会保证高优先级的 pod 被成功调度和运行。当 pod 调度不成功时，低优先级的 pod 会被驱逐出去，腾出空间给高优先级的 pod 使用。在 1.9 及以后的版本，Priority 同时会影响调度的顺序以及资源不足时 pod 被驱逐的顺序。

Pod优先级和抢占在Kubernetes 1.11处于beta阶段和在Kubernetes 1.14中达到GA阶段。自1.11以来，默认情况下已启用。

在Kubernetes旧版本中，Pod优先级和抢占仍然是alpha级别的功能，您需要明确启用它。要在旧版本的Kubernetes中使用这些功能，请按照Kubernetes版本的文档中的说明进行操作，方法是转到Kubernetes版本的文档存档版本。

| Kubernetes Version | Priority and Preemption State | Enabled by default |
| :----------------- | :---------------------------- | :----------------- |
| 1.8                | alpha                         | no                 |
| 1.9                | alpha                         | no                 |
| 1.10               | alpha                         | no                 |
| 1.11               | beta                          | yes                |
| 1.14               | stable                        | yes                |

### 最佳实践

在Kubernetes集群中，通过非DeamonSet方式（比如Deployment）部署关键服务时，为了在集群资源不足时仍能保证抢占调度成功

- 开启 *apiserver、controller-manager、scheduler，kubelet* 的 `--feature-gates=“PodPriority=true,ExperimentalCriticalPodAnnotation=true”`
- 配置 apiserver 的 runtime-config参数`--runtime-config=batch/v2alpha1=true` 及 admission-control 参数：`--admission-control=...,MutatingAdmissionWebhook,ValidatingAdmissionWebhook`
- 给Pod或者deployment的添加了ExperimentalCriticalPodAnnotation `scheduler.alpha.kubernetes.io/critical-pod=""`
- 部署在`kube-system` namespace
- 没有自定义PriorityClass



参考文档：

https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/
https://kubernetes.io/zh/docs/concepts/overview/kubernetes-api/
https://k8smeetup.github.io/docs/reference/api-overview/
https://kubernetes.io/docs/concepts/configuration/pod-priority-preemption/
