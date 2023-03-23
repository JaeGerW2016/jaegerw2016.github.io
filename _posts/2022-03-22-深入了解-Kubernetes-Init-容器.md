---
layout:     post
title:     深入了解 Kubernetes Init 容器
subtitle: 	init 容器是一种具有一些修改的操作行为和规则的容器
date:       2022-03-11
author:     J
catalog: true
tags:
    - Docker
    - Kubernetes
---



# 深入了解 Kubernetes Init 容器

Kubernetes 及其生态系统的受欢迎程度像滚雪球一样滚下珠穆朗玛峰。想象一下推动 Kubernetes 开发的设计模式、众多工作负载要求、工作负载类型和行为。每个版本都有很长的路要走以满足要求的增强功能。每个功能增强都打开了可能性。

其中之一是[init 容器功能](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/)。它在工作负载的初始化阶段带来了许多机会。

如果你想要一些关于 init 容器的实践时间，这里有一个[k8s Init Containers git repo](https://github.com/loft-sh/k8s-init-containers)。有了这个 repo，你可以在几分钟内部署 init 容器。

## 什么是初始化容器？

init 容器是一种具有一些修改的操作行为和规则的容器。最主要的特性之一是 init 容器在应用程序容器之前启动和终止，并且它们必须成功运行到完成。它们专门用于初始化工作负载环境。

假设我们手头有一个应用程序，需要在部署时进行一些设置。在 init 容器中完成设置任务将确保环境在应用程序启动之前准备就绪，从而实现我们拥有精简应用程序容器映像的目标。

执行这些设置任务的另一种方法是在应用程序容器上使用一些工具，例如 shell 脚本。在运行应用程序本身之前，该脚本将确保资源已创建且可用、执行磁盘操作、读取配置映射等。

## 使用 init 容器的动机

如果应用程序需要在初始化时运行任务，使用 distroless 容器镜像作为基础镜像几乎可以确保使用 init 容器。由于我们无法将工具安装到应用程序容器镜像中，因此我们需要创建一个新镜像，以便仅用于初始化以供新的 init 容器使用。

使用 init 容器的其他原因包括安全性改进，例如在应用程序运行之前运行特权任务、使用运行时不可用的秘密生成配置、资源可用性检查、数据库迁移等。基本上任何需要在之前完成的任务应用程序运行属于此类。以下是一些实际示例的列表：

- 等待所需资源准备就绪：
  - Kubernetes Services(+end points)
  - Database
  - Cache
  - Queue
  - API
  - Etc.
- 数据库迁移/应用种子
- 通过将特权任务移动到初始化容器来减少秘密/凭证暴露。
- 从需要凭据的 git 存储库或云存储中获取资产。
- 对部署时和运行时使用不同的安全上下文。
- 使用运行时属性生成配置文件，例如 pod IP、分配的外部 IP、主机名、外部入口 IP 等。
- 使用键/值存储生成配置文件。

## 初始化容器的生命周期

这是从通过 kubectl 应用清单到 pod 接收流量的生命周期概述。

![Pod 生命周期](https://loft.sh/blog/images/content/pod-init-to-ready.png?nf_resize=fit&w=1040)

## 初始化容器的特性和行为

Init 容器是一种容器类型，具有针对 Pod 初始化的特定差异。在确保存储和网络服务启动后，kubelet 在 pod 中的任何其他容器之前执行 init 容器。如果有多个 init 容器，它们会按照它们在配置中出现的顺序依次运行。

kubelet 期望 init 容器运行完成；为了让 kubelet 启动下一个 init 容器，当前的 init 容器必须成功运行完成。kubelet 重新启动失败的 init 容器，直到成功。

### 顺序运行

每个 init 容器必须以退出代码 0 成功完成才能运行下一个 init 容器（如果已定义）。如果任何 init 容器返回非零退出代码，kubelet 将重新启动它。一旦所有 init 容器都成功运行完成（零退出代码），kubelet 将并行启动常规容器。

### 重启政策

非零退出代码会根据[pod](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/#Pod)触发初始化容器的重新启动。[规格](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/#PodSpec)_ [重启策略](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/#lifecycle)。如果 pod 的 restartPolicy 设置为“Always”，init 容器将重新启动，就像 restartPolicy 设置为“OnFailure”一样。如果一个 pod 的 restartPolicy 设置为“从不”，那么 init 容器不会在失败时重新启动，因此该 pod 被视为永久失败。

[在PodSpec](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/#PodSpec)中更新 init 容器的 image 字段的值不需要重新启动 pod。

### 更新 init 容器镜像

用于 init 容器的映像的更改不保证重新启动或更改 pod。Kubelet 在 pod 的下一个周期中使用更新的镜像。

### 不支持的字段

[尽管 init 容器和普通容器共享容器](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/#Container)对象的确切规格，但不允许为 init 容器设置以下字段。

- [lifecycle](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/#lifecycle-1)
- [livenessProbe](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/#Probe)
- [readinessProbe](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/#Probe)
- [startupProbe](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/#Probe)

这些字段允许高级生命周期状态转换，这不适用于 init 容器；他们的成功仅由退出代码（exit code）定义。如果设置了这些值，则 PodSpec 的验证将失败。

### 请求和限制计算

资源管理是集群最重要的工作之一。初始化容器需要资源才能运行完成，因此为资源设置请求和限制是有意义的。

在 Pod 调度期间使用的实际资源请求和限制称为有效资源请求/限制。当 pod 中存在 init 容器时，有效请求/限制的计算方式略有不同。

对于 init 容器，调度程序会遍历所有 init 容器并找出所有 init 容器中最重要的请求或限制。用于调度的最重要的值是有效的初始化请求/限制。

当然，Pod 是 Kubernetes 部署的最小单元。所以我们需要考虑 pod 的有效请求/限制。Pod 的有效请求/限制是通过比较有效的初始化请求/限制和所有常规容器的请求/限制的总和来计算的。在 pod 调度中选择并使用特定资源的最高资源请求/限制。

这种行为允许我们对 init 容器使用不同的资源请求/限制，这非常好。但是，同时它们也很纠结，我们需要记住，我们为一些只在应用启动时运行的容器分配资源，这可能会阻止调度程序找到可用的节点。

### 关注点分离

使用 init 容器方法意味着我们将拥有至少两个不同的容器镜像；一个用于 init 容器的映像，其中包括所有用于设置的工具，另一个是应用程序映像。拥有单独的图像允许我们将图像的所有权分配给不同的团队。

### 安全

使用 init 容器可以带来一些安全优势。假设该应用程序需要来自 git 存储库或云存储的一些资产。我们只能提供对 init 容器的访问权限，而不是授予对应用程序容器的机密/凭据的访问权限，因为它可能会被未经授权的一方访问。减少访问意味着秘密/凭证的暴露是短暂的并且不易获得。

另一个方面是，现在我们不需要安装仅在 pod 启动时才会使用的工具。减少包和工具的数量可以减少攻击面。通过将启动时所需的工具移至 init 容器，您可以为您的应用程序使用 distroless 映像并让 init 容器处理这些任务。

## 配置初始化容器

初始化容器在[pod](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/#Pod)中定义。[规格](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/#PodSpec)_ [initContainers](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/#containers)数组，而常规容器定义在[pod](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/#Pod)下。[规格](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/#PodSpec)_ [容器](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/#containers)数组。两者都持有[Container](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/#Container)对象。

[吊舱](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/#Pod)_ [规范](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/#PodSpec)在下面的 Kubernetes 源代码中定义；我们可以看到 InitContainers 和 Containers 是[Container](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.23/#container-v1-core)类型的数组。

```go
// File: https://github.com/kubernetes/kubernetes/blob/e6c093d87ea4cbb530a7b2ae91e54c0842d8308a/pkg/apis/core/types.go#L2813
...
// PodSpec is a description of a pod
type PodSpec struct {
Volumes []Volume
// List of initialization containers belonging to the pod.
InitContainers []Container
// List of containers belonging to the pod.
Containers []Container
...
```

从配置上看，还是有细微差别的；如上所述，init 容器不支持[lifecycle](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/#lifecycle-1)、[livenessProbe、readinessProbe、startupProbe字段。](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/#Probe)

## init容器的调试和故障排除

我们可以直接或间接地收集有关 init 容器的信息。在我们深入了解各个初始容器状态之前，让我们看一下 pod 范围的指标。

### Pod状态

在 Pod 的初始化过程中，Pod status 上会显示一些“init container”相关的状态。这些有价值的输出以“init：”为前缀显示

| pod状态                    | 描述                                                         |
| -------------------------- | ------------------------------------------------------------ |
| init：N/M                  | 到目前为止，共有 M 个 init 容器中的 N 个 init 容器成功执行到完成。 |
| init：Error                | Init Container 已经并且一直在失败，并且 kubelet 会触发退避算法。 |
| init：CrashLoopBackOff     | Init Container 已经并且一直在失败，并且 kubelet 会触发退避算法。 |
| Pending                    | 初始化容器执行尚未开始。                                     |
| PodInitializing 或 Running | 所有初始化容器都以零退出代码执行完成。                       |

让我们通过几个例子来看看这些状态。

```other
kubectl get pods

NAME                                          READY   STATUS     RESTARTS   AGE
...
k8s-init-containers-668b46c54d-kg4qm          0/1     Init:1/2   1          8s
```

`Init:1/2`status 告诉我们有两个初始化容器，其中一个已经运行完成。

```other
kubectl get pods

NAME                                          READY   STATUS                  RESTARTS   AGE
...
k8s-init-containers-668b46c54d-kg4qm          0/1     Init:CrashLoopBackOff   5          4m12s
```

`Init:CrashLoopBackOff`会指出其中一个 init 容器反复失败。此时，kubelet 会重启失败的 init 容器，直到成功完成。

让我们使用`kubectl get pods`带有`--watch`flag 的命令来更清楚地查看 pod 状态的变化。此标志使命令能够观察对象并输出更改。

```other
kubectl get pods --watch

NAME                                 READY   STATUS                  RESTARTS   AGE
k8s-init-containers-64f984c8d7-tdjrg 0/1     Pending                 0          0s
k8s-init-containers-64f984c8d7-tdjrg 0/1     Pending                 0          4s
k8s-init-containers-64f984c8d7-tdjrg 0/1     Init:0/4                0          4s
k8s-init-containers-64f984c8d7-tdjrg 0/1     Init:0/4                0          6s
k8s-init-containers-64f984c8d7-tdjrg 0/1     Init:Error              0          7s
k8s-init-containers-64f984c8d7-tdjrg 0/1     Init:1/4                1          8s
k8s-init-containers-64f984c8d7-tdjrg 0/1     Init:1/4                1          9s
k8s-init-containers-64f984c8d7-tdjrg 0/1     Init:2/4                1          43s
k8s-init-containers-64f984c8d7-tdjrg 0/1     Init:3/4                1          44s
k8s-init-containers-64f984c8d7-tdjrg 0/1     PodInitializing         0          45s
k8s-init-containers-64f984c8d7-tdjrg 1/1     Running                 0          46s
```

起初，pod 处于`Pending`集群接受 pod 的状态；但是，它还没有被安排。我们可以看到有四个init容器（`Init:x/4`）；接下来，依次启动 init 容器。其中一个 init 容器出现错误 ( `Init:Error`)，并且已重新启动 ( `RESTARTS: 1`)。重启后，本次init容器执行成功。下一个 init 容器启动，其余的 init 容器也运行成功完成。pod 状态从状态移动`PodInitializing`到“运行”状态。

### Pod条件

pod 条件是关于某些状态的 pod 的高级状态视图，这是一个很好的起点。kubelet 填充[Pod](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/#Pod)。[状态](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/#PodStatus)。[带有PodCondition](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.23/#podcondition-v1-core)对象的[条件](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-conditions)数组。

有四种类型的条件，每一种都有一个状态字段。

- Initialized
- Ready
- ContainersReady
- PodScheduled

status 字段的值可以是`True`, `False`, `Unknown`, 表示 Pod 是否通过条件要求。

从 init 容器的角度来看，最相关的条件类型是“Initialized”。一旦每个 init 容器成功运行完成，Pod 的“Initialized”条件的状态设置为`True`，表示初始化成功完成。

```other
...
Conditions:
  Type              Status
  Initialized       True 
  Ready             True 
  ContainersReady   True 
  PodScheduled      True 
...
```

当 init 容器出现故障时，这是一个条件数组。由于初始化尚未完成，其余容器未运行。Kubelet 重新启动失败的 init 容器。

```other
...
Conditions:
  Type              Status
  Initialized       False
  Ready             False
  ContainersReady   False
  PodScheduled      True
...
```

您可以收集 pod 名称列表及其 pod 状态条件以进行编程访问，其类型为 ，`Initialized`具有以下[JSONPath](https://kubernetes.io/docs/reference/kubectl/jsonpath/)表达式。

```other
kubectl get pods -o=jsonpath='{"Pod Name, Condition Initialized"}{"\n"}{range .items[*]}{.metadata.name},{@.status.conditions[?(@.type=="Initialized")].status}{"\n"}{end}'

Pod Name, Condition Initialized
buildkit-builder0-76784c68c7-c52nw,True
k8s-init-containers-b4ddb8ffc-2sh4m,True
postgresql-postgresql-0,True
redis-master-0,True
```

### ContainerStatus 对象（在 pod.status.initContainerStatuses[] 中）

可以在[pod](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/#Pod)下找到 init 容器的状态。[status](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/#PodStatus) .initContainerStatuses 数组，它为每个 init 容器保存一个[ContainerStatus对象。](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.23/#containerstatus-v1-core)

```yaml
kubectl get pods k8s-init-containers-64f984c8d7-tdjrg -o yaml
...
initContainerStatuses:
  - containerID: containerd://b60870084a2065d…
    image: docker.io/loftsh/app-init:FLiBwNW
    imageID: docker.io/loftsh/app-init@sha256:e46e2…
    lastState: {}
    name: init-fetch-files
    ready: true
    restartCount: 2
    state:
    terminated:
    containerID: containerd://b60870084a2065def2d289f7…
    exitCode: 0
    finishedAt: "2022-02-27T02:09:37Z"
    reason: Completed
    startedAt: "2022-02-27T02:09:37Z"
...
```

我们可以看到“init-fetch-files”init容器已经重启了两次；在最后一次重新启动中，它通过以零退出代码退出来完成。

## 容器状态字段

我们在[ContainerStatus](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.23/#containerstatus-v1-core)`kubectl describe pods <pod-name>`中看到的状态字段在命令输出中也可用 。让我们检查一个包含更多 init 容器的 pod。在这个例子中，我们使用一个带有两个 init 容器和一个 app 容器的 pod。

为了收集更多信息，让我们检查一下`kubectl describe pod <pod-name>`命令的输出。

```other
kubectl describe pod k8s-init-containers-668b46c54d-kg4qm
```

为简洁起见，一些输出被剪切，代码块有省略号来表示剪切。

查看第一个 init 容器的输出，“init-fetch-files”。Container 的状态为`Terminated`，原因为 Completed，Exit Code 为零。容器在第一次尝试时以零代码执行并退出（`Restart Count`为零）。

```other
…
Init Containers:
  init-fetch-files:
    Container ID:  containerd://31ac3…
    Image:         loftsh/app-init:rQyZJBK
    Image ID:      docker.io/loftsh/app-init@sha256:79ce…
    Port:          <none>
    Host Port:     <none>
    Command:
      fetchFiles.sh
    State:          Terminated
      Reason:       Completed
      Exit Code:    0
      Started:      Tue, 15 Mar 2022 13:54:39 +0000
      Finished:     Tue, 15 Mar 2022 13:54:39 +0000
    Ready:          True
    Restart Count:  0
    Environment:
      PUBLIC_ROOT:  /data/public
    Mounts:
      /data from app-data (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-44vbg (ro)
…
```

让我们检查第二个初始化容器“init-check-services”；状态是`Waiting`，原因是`CrashLoopBackOff`，并且`Exit Code`是一，非零值表示错误。字段显示`Restart Count`容器重启223次；显然，有问题。

```other
…
Init Containers:
…
  init-check-services:
    Container ID:  containerd://eb1dbae93999a63fed7…
    Image:         loftsh/app-init:kxPrHjW
    Image ID:      docker.io/loftsh/app-init@sha256:5…
    Port:          <none>
    Host Port:     <none>
    Command:
      checkServices.sh
    State:          Waiting
      Reason:       CrashLoopBackOff
    Last State:     Terminated
      Reason:       Error
      Exit Code:    1
      Started:      Tue, 15 Mar 2022 14:11:23 +0000
      Finished:     Tue, 15 Mar 2022 14:11:23 +0000
    Ready:          False
    Restart Count:  223
    Environment:
      APP_NAME:  app
      SERVICES:  redis-master,postgresql
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-ftmj6 (ro)
```

知道下一个容器的状态是什么吗？

最后，让我们看看应用容器状态。状态为Waiting，原因设置为PodInitializing；由于并非所有 init 容器都成功运行完成，因此 pod 仍处于初始化状态。

```other
…
Containers:
  app:
    Container ID:   
    Image:          loftsh/app:sNKfQGb
    Image ID:       
    Port:           <none>
    Host Port:      <none>
    State:          Waiting
      Reason:       PodInitializing
    Ready:          False
    Restart Count:  0
    Environment:
      APP_NAME:  app
    Mounts:
      /data from app-data (ro)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-44vbg (ro)
```

Pod条件也已适当设置。

```other
…
Conditions:
  Type              Status
  Initialized       False 
  Ready             False 
  ContainersReady   False 
  PodScheduled      True 
…
```

您可能已经注意到每个容器状态的 State 字段的输出不同；容器状态及其字段分为三种类型：

- [ContainerStateRunning](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.23/#containerstaterunning-v1-core)
  - startedAt
- [ContainerStateTerminated](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.23/#containerstateterminated-v1-core)
  - containerID
  - exitCode
  - finishedAt
  - message
  - reason
  - signal
  - startedAt
- - [ContainerStateWaiting](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.23/#containerstatewaiting-v1-core)
    - message
    - reason

如果记录了多个状态，则`Last State`添加，这有助于故障排除。Last State 的示例如下：

```other
...
Last State: Terminated
Reason: Error
Exit Code: 1
Started: Fri, 04 Mar 2022 07:16:21 +0000
Finished: Fri, 04 Mar 2022 07:16:21 +0000
...
```

输出显示容器以非零退出代码 (1) 终止。

### 容器Log日志

您可以像访问常规容器一样访问初始化容器日志。

```other
kubectl logs k8s-init-containers-668b46c54d-kg4qm -c init-check-services

At least one (app-redis) of the services (app-redis app-postgresql app-mongodb) is not available yet.
```

### Event事件日志

Kubernetes 事件也是一个很好的信息来源。

```other
kubectl get events

LAST SEEN   TYPE      REASON      OBJECT                                               MESSAGE
81s         Warning   BackOff     pod/k8s-init-containers-5c694cd678-gr8zg Back-off    restarting the failed container
```

## 结论

Init 容器为应用程序或服务的初始化阶段带来了不同的思维方式。正如我们在前面的部分中所看到的，使用 init 容器有很多好处，

包括

- 增强安全性、所有权分离、

- 保持应用程序容器尽可能精简等。

  配置、故障排除和监控 init 容器与常规容器没有太大区别，除了您需要考虑的一些行为差异。

## 参考文献和进一步阅读

- [Kubernetes Documentation/Concepts - Init Containers](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/)
- [Kubernetes Documentation/Tasks/Configure Pods and Containers - Configure Pod Initialization](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-initialization/)
- [Kubernetes Documentation/Tasks/Monitoring, Logging, and Debugging - Debug Init Containers](https://kubernetes.io/docs/tasks/debug-application-cluster/debug-init-containers/)
- [Kubernetes Documentation/Concepts/Workloads/pods - Pod Lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle#pod-and-container-status)
- [Kubernetes Documentation/Reference/Kubernetes API/Workload Resources/Pod](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/)
- [Kubernetes Documentation/Reference/API - Pod](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.23/#pod-v1-core)
- [Kubernetes Documentation/Reference/API - PodSpec](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.23/#podspec-v1-core)
- [Kubernetes Documentation/Reference/API - PodStatus](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.23/#podstatus-v1-core)
- [Kubernetes Documentation/Reference/API - Container](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.23/#container-v1-core)
- [Kubernetes Documentation/Reference/API - ContainerStateTerminated](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.23/#containerstateterminated-v1-core)
- [K8s Init Containers git repo](https://github.com/loft-sh/k8s-init-containers)