---
layout:     post
title:     如何为 Trivy 编写自定义策略
subtitle:  How to write custom policies for Trivy
date:       2023-02-27
author:     J
catalog: true
tags:
    - Trivy
    - Kubernetes
---
## 前言
在之前的文章中[在 Jenkins 中使用 Trivy 进行容器扫描](https://jaegerw2016.github.io/2023/02/20/Container-Scanning-with-Trivy-in-Jenkins/)我们已经了解到Trivy 最基础的扫描镜像安全风险漏洞的功能,其实Trivy的功能远不止这些,今天我们着重来了解下Trivy根据自定义策略来扫描Kubernetes的配置文件.

目前Trivy支持以下类型的配置文件：
- Kubernetes
- Dockerfile, Containerfile
- Terraform
- CloudFormation
- Helm Chart
- RBAC

## 编写您的第一个自定义策略
您需要记住的最重要的一点是自定义策略是Trivy用Rego. 我强烈建议您熟悉[Rego语言](https://www.openpolicyagent.org/docs/latest/policy-language/)。

> Rego语言最初由Datalog开发,作为OPA或者Policy的默认语言,后期Kyverno也用了这种声明式语言

如下显示，Trivy支持特定配置文件并通过配置类型的文件扩展名检测正确的策略。

| File extension                                    | Configuration              |
| ------------------------------------------------- | -------------------------- |
| .yaml, .yml and *.json                            | Kubernetes / Helm          |
| Dockerfile, Dockerfile., and .Dockerfile          | Dockerfile                 |
| Containerfile, Containerfile., and .Containerfile | Containerfile              |
| .yaml, .yml and *.json                            | CloudFormation             |
| .tf and .tf.json                                  | Terraform / Terraform Plan |

### 自定义策略剖析

我将使用一个非常简单的策略用例来解释自定义策略的结构。

让我们假设以下场景：您只想允许将来自特定容器注册表的 Pod 部署到您的集群。

```
package user.kubernetes.ID001

import future.keywords.in
import data.lib.kubernetes

default allowedRegistries = ["quay.io","ghcr.io","gcr.io"]

__rego_metadata__ := {
  "id": "ID001",
  "title": "Allowed container registry checks",
  "severity": "CRITICAL",
  "description": "The usage of non allowed container registries is not allowed",
}

__rego_input__ := {
  "combine": false,
  "selector": [{"type": "kubernetes"}],
}

allowedRegistry(image) {
  registry := allowedRegistries[_]
  startswith(image, registry)
}

deny[msg] {
  container := kubernetes.containers[_]
  not allowedRegistry(container.image)
  msg :=  kubernetes.format(sprintf("Container '%s' of %s '%s' comes from not approved container registry", [container.name, kubernetes.kind, kubernetes.name]))
}
```
`package` (必需字段)

- 必须遵循 Rego 的规范
- 根据政策必须是唯一的
- 应该包括唯一性的策略 ID
- 可以包括组名，例如kubernetes为了清楚起见
- 组名对策略评估没有影响

`__rego_metadata__`（可选字段）
使用有用的信息帮助丰富Trivy扫描结果。所有字段都是可选的。请查看官方文档以获取有关所有可用字段及其含义的更多信息

`__rego_input__`（可选字段）
允许过滤输入数据的可选字段。在我的示例中，我只想扫描*Kubernetes资源*并忽略任何其他配置类型。

`deny`（必需字段）
- 应该是deny或开始于deny_
  - 根据 AquaSecurity 的说法warn，violation它也适用于兼容性，但deny建议使用。您始终可以severity在__rego_metadata__
- 应该返回string
  - 虽然objectwithmsg字段是可以接受的，但是其他的字段会被丢弃，string推荐使用。
  - 例如{"msg": "deny message", "details": "something"}

`  import future.keywords`是一个特殊的导入，允许在您的策略中使用未来的关键字。

`  import data.lib.result`是一个特殊的导入，允许使用result库来突出显示结果。

接下来我们来看下`deny`这个结构体中写的是什么?

首先，我们检查我们是否只在 type 上应用规则Pod。然后我们遍历Pod中的所有容器，Pod并检查容器的镜像是否来自开头`allowedContainerRegistry`中的镜像仓库中。

## 验证

trivy 扫描到`Pod.yaml`中`nginx`的镜像并非来自`"quay.io","ghcr.io","gcr.io"`这几个仓库.

```
root@ubuntu:~/trivy-custom-policies/pod# trivy conf --severity HIGH,CRITICAL --policy ./policy --namespaces user ./config
2023-03-01T11:51:48.904+0800    INFO    Misconfiguration scanning is enabled
2023-03-01T11:51:51.065+0800    INFO    Detected config files: 1

pod.yaml (kubernetes)

Tests: 68 (SUCCESSES: 67, FAILURES: 1, EXCEPTIONS: 0)
Failures: 1 (HIGH: 0, CRITICAL: 1)

CRITICAL: Container 'nginx' of Pod 'nginx' comes from not approved container registry
═══════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════
The usage of non allowed container registries is not allowed
───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────

root@ubuntu:~/trivy-custom-policies/pod#

```
