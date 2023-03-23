---
title: Kubernetes附加组件可实现更高效的计算
date: 2019-01-03 02:56:37
tags:
---



如果说“启动” [Kubernetes](https://akomljen.com/tag/kubernetes/)集群是一项相对容易的工作。部署应用程序以在Kubernetes之上工作需要更多的努力，特别是如果您不熟悉容器。对于使用Docker的人来说，这也是一项相对简单的工作，但当然，您需要掌握Helm等新工具。然后，当你把所有的东西放在一起，当你试图在生产中运行你的应用程序时，你会发现有很多缺失的部分。可能Kubernetes做的不多，对吧？好吧，Kubernetes是可扩展的，并且有一些插件或附件可以让您的生活更轻松。

>原文档：https://akomljen.com/kubernetes-add-ons-for-more-efficient-computing/
### 什么是Kubernetes附加组件？

简而言之，*附加组件扩展了Kubernetes的功能*。它们中有很多，很有可能你已经使用了一些。例如，网络插件或CNI，如Calico或Flannel，或CoreDNS（现在是默认的DNS管理器），或着名的Kubernetes仪表板。我说有名，因为这可能是集群运行后你将尝试部署的第一件事:)。上面列出的是一些核心组件，CNI必须具有，DNS相同，以使您的集群正常运行。但是，一旦开始部署应用程序，您就可以做更多事情。输入Kubernetes附加组件以实现更高效的计算！

##### 如果您是Kubernetes的新手，请先阅读本书[Kubernetes in Action](https://amzn.to/2CJO1mb)

### Cluster Autoscaler - CA.

**[Cluster Autoscaler](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler)根据利用率扩展您的群集节点。**如果您有待处理的pod，CA将扩展群集，如果未充分利用节点，则将其缩小 - 默认设置为`0.5`并可配置`--scale-down-utilization-threshold`。你绝对不希望pod处于挂起状态，同时，你不想运行未充分利用的节点 - 浪费金钱！

**使用案例：** AWS集群中有两个实例组或自动调节组。它们在两个可用区域1和2中运行。您希望根据利用率扩展群集，但是您希望在两个区域中具有相似数量的节点。此外，您还希望使用CA自动发现功能，因此您无需在CA中定义最小和最大节点数，因为这些节点已在自动缩放组中定义。并且您希望在主节点上部署CA.

以下是通过Helm的CA示例安装，以匹配上述用例：

```
⚡ helm install --name autoscaler \
    --namespace kube-system \
    --set autoDiscovery.clusterName=k8s.test.akomljen.com \
    --set extraArgs.balance-similar-node-groups=true \
    --set awsRegion=eu-west-1 \
    --set rbac.create=true \
    --set rbac.pspEnabled=true \
    --set nodeSelector."node-role\.kubernetes\.io/master"="" \
    --set tolerations[0].effect=NoSchedule \
    --set tolerations[0].key=node-role.kubernetes.io/master \
    stable/cluster-autoscaler
```

您需要进行一些其他更改才能实现此目的。有关更多详细信息，请查看此帖子 - [AWS上的Kubernetes Cluster Autoscaling](https://akomljen.com/kubernetes-cluster-autoscaling-on-aws/)。

### Horizo​​ntal Pod Autoscaler - HPA

***[Pod水平自动伸缩](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)自动缩放基于观察到的CPU使用率在复制控制器，部署或副本集豆荚的数目。通过[自定义指标](https://git.k8s.io/community/contributors/design-proposals/instrumentation/custom-metrics-api.md)支持，还可以使用其他一些应用程序提供的指标。***

HPA在Kubernetes世界并不是什么新东西，但Banzai Cloud最近发布了[HPA Operator](https://github.com/banzaicloud/hpa-operator)，简化了它。您需要做的就是为您的部署或StatefulSet提供注释，HPA操作员将完成剩下的工作。在[这里](https://github.com/banzaicloud/hpa-operator#annotations-explained)查看支持的注释。

使用Helm安装HPA操作员非常简单：

```
⚡ helm repo add akomljen-charts https://raw.githubusercontent.com/komljen/helm-charts/master/charts/

⚡ helm install --name hpa \
    --namespace kube-system \
    akomljen-charts/hpa-operator

⚡ kubectl get po --selector=release=hpa -n kube-system
NAME                                  READY     STATUS    RESTARTS   AGE
hpa-hpa-operator-7c4d47dd4-9khpv      1/1       Running   0          1m
hpa-metrics-server-7766d7bc78-lnhn8   1/1       Running   0          1m
```

部署Metrics Server后，您还可以使用`kubectl top pods`命令。监视pod的CPU或内存使用情况可能很有用！;）

HPA可以从一系列的API聚集（的取度量`metrics.k8s.io`，`custom.metrics.k8s.io`和`external.metrics.k8s.io`）。但是，通常，HPA将使用`metrics.k8s.io`Heapster提供的API（不再使用Kubernetes 1.11）或Metrics Server。

向部署添加注释后，您应该能够使用以下方法对其进行监控：

```
⚡ kubectl get hpa
NAME       REFERENCE             TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
test-app   Deployment/test-app   0%/70%    1         4         1          10m
```

请记住，您在上面看到的CPU目标是基于此特定pod的已定义CPU请求，而不是节点上可用的总CPU。

### Addon Resizer

[Addon resizer](https://github.com/kubernetes/autoscaler/tree/master/addon-resizer)是一个有趣的插件，您可以在上面的场景中使用Metrics Server。在将更多pod部署到群集时，最终Metrics Server将需要更多资源。**Addon resizer容器监视Deployment（例如Metrics Server）中的另一个容器，并向上和向下垂直缩放依赖容器。**Addon resizer可以根据节点数线性扩展Metrics Server。有关详细信息，请查看官方文档。

### Vertical Pod Autoscaler - VPA

您需要为将在Kubernetes上部署的服务定义CPU和内存请求。如果您没有将默认CPU请求设置为`100m`或`0,1`可用CPU。资源请求有助于`kube-scheduler`确定运行特定pod的节点。但是，很难定义适合更多环境的“足够好”的值。**[Vertical Pod Autoscaler](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler)根据pod使用的资源[自动](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler)调整CPU和内存请求。**它使用Metrics Server获取pod指标。请记住，您仍需要手动定义资源限制。

我不会在这里详述，因为VPA确实需要专门的博客文章，但有一些事情你应该知道：

*   VPA仍然是一个早期项目，所以要注意
*   您的群集必须支持`MutatingAdmissionWebhooks`，默认情况下自Kubernetes 1.9启用
*   它不能与HPA一起使用
*   当资源请求更新时，它将重新启动所有pod，这是预期的类型

### Descheduler

这`kube-scheduler`是负责Kubernetes调度的组件。但是，由于Kubernetes的动态性，有时pod可能会在错误的节点上结束。您可能正在编辑现有资源，添加节点亲和性或（反）pod亲和性，或者您在某些服务器上有更多负载，而某些服务器几乎在空闲时运行。一旦pod运行，`kube-scheduler`将不会尝试再次重新安排它。根据环境的不同，您可能会有很多活动部件。

**[Descheduler会](https://github.com/kubernetes-incubator/descheduler)检查可以移动的pod，并根据定义的策略将其驱逐出去。**Descheduler不是默认的调度程序替代品，取决于它。该项目目前在Kubernetes孵化器中尚未准备好生产。但是，我发现它非常稳定并且效果很好。Descheduler将作为CronJob在您的集群中运行。

我写了一篇专门的文章[Meet a Kubernetes Descheduler](https://akomljen.com/meet-a-kubernetes-descheduler/)，你应该查看更多细节。

### k8s Spot Rescheduler

我试图解决在AWS上管理多个自动扩展组的问题，其中一个组是按需实例，另一个竞争实例。问题是，一旦您扩展了竞争实例，您想要从按需实例移动容器，以便您可以将其缩小。**[k8s spot recheduler](https://github.com/pusher/k8s-spot-rescheduler)试图通过将pod驱逐到可用的点来减少on-demand实例的负载。** *实际上，重新调度程序可用于将任何节点组的负载移除到不同的节点组上。他们只需要贴上适当的标签。*

我还创建了一个Helm图表，以便于部署：

```
⚡ helm repo add akomljen-charts https://raw.githubusercontent.com/komljen/helm-charts/master/charts/

⚡ helm install --name spot-rescheduler \
    --namespace kube-system \
    --set image.tag=v0.2.0 \
    --set cmdOptions.delete-non-replicated-pods="true" \
    akomljen-charts/k8s-spot-rescheduler
```

对于一个完整列表`cmdOptions`检查[这里](https://github.com/pusher/k8s-spot-rescheduler#flags)。

要使k8s spot recheduler正常工作，您需要标记您的节点：

*   按需节点 - `node-role.kubernetes.io/worker: "true"`
*   点节点 - `node-role.kubernetes.io/spot-worker: "true"`

并`PreferNoSchedule`在点播实例上添加污点，以确保k8s点重定时器在做出调度决策时更喜欢点。



