---
layout:     post
title:      在Kubernetes上安全地构建容器镜像
subtitle:   运行容器并推拉容器镜像不再需要Docker
date:       2020-04-07
author:     J
catalog: true
tags:
    - kubernetes
---


在我们安全地在Kubernetes上构建和推拉容器镜像之前，我想分享一些有关容器术语的想法。我通常将在Kubernetes Pod中作为容器运行的镜像称为**容器镜像**而不是**Docker镜像**。

早在2015年，Docker公司便将[Docker image 格式](https://blog.docker.com/2017/07/oci-release-of-v1-0-runtime-and-image-format-specifications/)捐赠给了当时刚刚成立的[Open Container Initiative（OCI）](https://www.opencontainers.org/)。除了容器[Image格式规范外](https://github.com/opencontainers/image-spec/blob/master/spec.md)，OCI还维护用于容器执行的开放[Runtime规范](https://github.com/opencontainers/runtime-spec/blob/master/spec.md)。这意味着*不再需要**Docker*** 来运行容器和提取容器镜像-而且，正如您将在本文后面所看到的，甚至*不需要*构建和推送容器镜像。

要了解使用DinD生成和推送容器图像的弊端，我再次建议您阅读[JérômePetazzoni的文章](https://jpetazzo.github.io/2015/09/03/do-not-use-docker-in-docker-for-ci/)。快速摘要：

- 安全性— DinD容器必须在`--privileged`启用标志的情况下运行，从而导致对运行Docker引擎的主机的不良攻击。
- 性能-使用临时`DinD`容器时，层之间的缓存不会在构建之间共享。每次使用新的DinD容器时，必须拉出所有图像的所有层，这会导致构建容器图像的速度大大降低。

因此，我们已经确定将DinD用于CI / CD，尤其是用于构建和推送容器镜像是一个坏主意。但是对于Kubernetes CD，如果要安装用于CI / CD的Docker套接字，还应该三思而后行-包括构建和推送容器镜像。

##  性能

让我们暂时搁置安全性。事实证明，在基于Kubernetes的CD环境中，挂载Docker Socket存在重大问题。

Kubernetes集群由一个或多个Node组成，并且Kubernetes在这些Node上调度和运行[Pods](https://kubernetes.io/docs/concepts/workloads/pods/pod/)。将Docker套接字挂载到Pod时，会将`/var/run/docker.sock`文件挂载到组成Pod的每个容器中。当这些容器针对该套接字运行Docker命令时，它们实际上实际上是由以**root**身份运行的Docker守护程序直接执行的在计划Pod的Node上。

Kubernetes调度程序无法跟踪这些其他容器是否正在运行-它们不是由Kubernetes管理的，而是由在Pod被调度的节点上运行的Docker守护程序管理的。这可能会导致严重的Kubernetes调度问题，尤其是在繁忙的CD群集上。首先使用Kubernetes的主要原因之一是由于其强大的容器编排和调度功能-那么为什么要为CI / CD规避呢？

另一个性能问题是容器镜像层缓存。如果您依赖Docker守护程序提供的内置缓存，那么您可能最终会在同一容器镜像或共享层的其他容器镜像的不同构建的不同K8s节点上落下，从而否定了Docker守护程序在特定的K8s工作节点。

此外，主机上存储的任何层以及通过Docker套接字运行的容器生成的任何日志都不会自动为您清除。

##  安全

简而言之，增强的安全性与减少攻击面或攻击媒介密切相关。Kubernetes上的CD没什么不同。如果Docker套接字暴露于CD作业，那么好奇/恶意的开发人员可以修改该作业以在构建步骤中运行Docker命令，从而有可能进入`root`该作业所到达的节点。如果他们以的身份访问基础主机`root`，则可以使用相当简单的方法来升级特权并获得对整个Kubernetes集群的访问权。许多群集操作员没有适当的监视以正确检测到此类活动并将其与合法的CD作业运行分开，因此，利用漏洞很可能会被忽视。

**DinD**一直[要求](https://blog.docker.com/2013/09/docker-can-now-run-within-docker/)`--privileged`[为运行DinD的容器启用](https://blog.docker.com/2013/09/docker-can-now-run-within-docker/)[该](https://blog.docker.com/2013/09/docker-can-now-run-within-docker/)[标志](https://blog.docker.com/2013/09/docker-can-now-run-within-docker/) —一直被认为是不安全的。但是安装Docker套接字从未如此安全，它依赖于使用专用Docker守护程序实例将CI / CD工作负载与其他容器工作负载（例如生产应用程序）隔离开来。

尽管对于仅使用CI / CD的单一用途群集而言，这种规模规模很大，但对于具有多个工作负载却应该隔离的K8s群集而言，规模规模却非常高。您应该期望将其与运行在K8s上的容器隔离。例如，**生产**容器可能只是远离CD作业运行的名称空间的名称空间。

如果走这条路，最终结果是*任何可以修改CD作业的人都可以成为整个集群的root*。

对于某些人来说，在Kubernetes上安装CD的Docker套接字的缺点非常明显，以至于他们[建议回到DinD](https://applatix.com/case-docker-docker-kubernetes-part-2/)。但是我们已经**不再使用DinD**和安装Docker套接字作为Kubernetes上CD的可接受方法了。在讨论替代方案之前，我们将探索K8s功能，这些功能允许您阻止在K8s集群（或集群的一部分）上运行的所有容器使用DinD和Docker套接字的安装。

通过将Kubernetes用于动态临时CD执行器，您可以直接在为Kubernetes管理的[Pod中](https://kubernetes.io/docs/concepts/workloads/pods/pod/)专门为执行该步骤而专门创建的容器中直接运行每个CD步骤。除了协调这些CD Pod的调度和运行之外，K8还允许管理其生命周期的其他方面，以包括Pod规范的安全敏感方面，从而实现Pod创建和更新的细粒度授权。

[Pod安全策略准入控制器](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#podsecuritypolicy)是用于管理[Pod安全](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#podsecuritypolicy)的本机K8s解决方案。可以通过以下方式配置Pod安全策略（PSP）：如果将其配置为安装Docker套接字，则不会安排使用正确配置的策略在Pod中运行的容器，也不允许将其作为容器运行。`--privileged`容器—禁用DinD。

##  使用Pod安全策略阻止DinD和挂载Docker socket

Pod安全策略准入控制器是[增强K8s群集安全性的关键功能](https://kubernetes.io/blog/2018/07/18/11-ways-not-to-get-hacked/#6-use-linux-security-features-and-podsecuritypolicies)。Pod安全策略准入控制器允许您指定Pod安全策略，以限制允许执行的容器。如果将Pod中的容器配置为执行Pod安全策略不允许的操作，则K8不会调度Pod。

[*来自Kubernetes官方文档*](https://kubernetes.io/docs/concepts/policy/pod-security-policy/#what-is-a-pod-security-policy)*：*

> Pod安全策略是控制Pod规范的安全敏感方面的群集级资源。PodSecurityPolicy对象定义了Pod必须运行的一组条件才能被系统接受，以及相关字段的默认值

让我们看一些可以大大减少*攻击*面，从而降低风险的特定设置：

- `privileged`：设置为`false`将会禁止使用DinD。
- `runAsUser`：将其设置为，以`MustRunAsNonRoot`使容器不能以`ROOT`用户身份运行。

*注意：您将需要允许*`*USER root*`*实际上使用Kaniko做任何有意义的事情来构建和推送容器镜像，因此您很可能需要设置*`*runAsUser*`*为*`*RunAsAny*`*。Kaniko PSP的目标是减少其他可用的攻击媒介。*

- `allowPrivilegeEscalation`：禁用特权升级，以使容器的子进程无法获得比其父进程更多的特权。
- `volumes`注意：不允许通过指定[特定的卷类型](https://kubernetes.io/docs/concepts/storage/volumes/)并将主机目录/文件作为卷挂载，并且不允许`hostPath`任何CD容器使用该卷。这将禁用挂载Docker套接字的功能。
- PSP `annotations`：通过注释将所有Pod容器限制在`runtime/default`**seccomp**配置文件中，`seccomp.security.alpha.kubernetes.io/defaultProfileName`并且不进行设置，`seccomp.security.alpha.kubernetes.io/allowedProfileNames`因此默认值无法更改。

这是一个限制性**PSP**的示例，该示例不允许**挂载Docker socket**，也不允许**DinD**：

```yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
name: cd-restricted
annotations:
seccomp.security.alpha.kubernetes.io/defaultProfileName:  'runtime/default'
apparmor.security.beta.kubernetes.io/defaultProfileName:  'runtime/default'
spec:
privileged: false
# Required to prevent escalations to root.
allowPrivilegeEscalation: false
# This is redundant with non-root + disallow privilege escalation,
# but we can provide it for defense in depth.
requiredDropCapabilities:
- ALL
# Allow core volume types. But more specifically, don't allow mounting host volumes to include the Docker socket - '/var/run/docker.sock'
volumes:
- 'configMap'
- 'emptyDir'
- 'projected'
- 'secret'
- 'downwardAPI'
# Assume that persistentVolumes set up by the cluster admin are safe to use.
- 'persistentVolumeClaim'
hostNetwork: false
hostIPC: false
hostPID: false
runAsUser:
# Don't allow containers to run as ROOT
rule: 'MustRunAsNonRoot'
seLinux:
# This policy assumes the nodes are using AppArmor rather than SELinux.
rule: 'RunAsAny'
supplementalGroups:
rule: 'MustRunAs'
ranges:
# Forbid adding the root group.
- min: 1
max: 65535
fsGroup:
rule: 'MustRunAs'
ranges:
# Forbid adding the root group.
- min: 1
max: 65535
readOnlyRootFilesystem: false
```

## 使用Containerd替代Docker

针对以上DinD和挂载主机Docker socket的安全风险，另一种引人注目的解决方法就是不使用Docker作为容器的引擎，而是用Containerd来做替换

[**Containerd**](https://containerd.io/)是上述OCI镜像运行时的一种实现（也是[ Docker捐赠给CNCF的](https://blog.docker.com/2017/03/docker-donates-containerd-to-cncf/)），并且（在[撰写](https://blog.docker.com/2017/03/docker-donates-containerd-to-cncf/)本文时）[得到Google Kubernetes Engine（GKE）的支持](https://cloud.google.com/kubernetes-engine/docs/concepts/using-containerd)。通过使用所提供的OCI runtime规范**containerd**，实际上并不需要Docker，而且Kubernetes与Containerd配合具有更好的性能。因为Docker实际也是在底层调用**containerd**，导致Docker额外的守护程序和不必要的通信开销。