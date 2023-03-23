---
layout:     post
title:     使用Trivy Operator 扫描Kubernetes集群安全
subtitle:  Scan Kubernetes cluster for security issues using trivy operator
date:       2023-03-08
author:     J
catalog: true
tags:
    - Trivy
    - Kubernetes
---
## 前言
在之前的文章中我们讨论了如何在`jenkins`的`CI/CD`流程的`Pipeline`中去集成`Trivy`以及如何给`Trivy`编写自定义策略来工作,那我们今天就来尝试下用`Trivy Operator`给`Kubernetes`集群来做一次安全扫描,并输出相应的`Grafana`图表

之前的2个文章
- [在 Jenkins 中使用 Trivy 进行容器扫描](https://jaegerw2016.github.io/2023/02/20/Container-Scanning-with-Trivy-in-Jenkins/)
- [如何为 Trivy 编写自定义策略](https://jaegerw2016.github.io/2023/02/27/How-to-write-custom-policies-for-Trivy/)

### Trivy Operator

Trivy-Operator 利用 trivy 安全工具，将其输出结果整合到 Kubernetes CRDs（自定义资源定义）中，并通过 Kubernetes API 使安全报告可访问。这样，用户可以以 Kubernetes 原生的方式查找和查看与不同资源相关的风险。

Trivy operator 根据 Kubernetes 集群的工作负载和其他变化自动更新安全报告，生成以下报告：

`漏洞扫描`：针对 Kubernetes 工作负载的自动漏洞扫描。
`配置审计扫描`：针对具有预定义规则或自定义 Open Policy Agent（OPA）策略的 Kubernetes 资源的自动配置审计。
`暴露的 Secret 扫描`：自动扫描可以查找并详细说明集群中暴露的 Secrets 的位置。
`RBAC 扫描`：基于角色的访问控制扫描提供有关安装的不同资源的访问权限的详细信息。
`K8s 核心组件基础设施评估扫描`: Kubernetes 基础设施核心组件（etcd、apiserver、scheduler、controller-manager 等）的设置和配置。
`合规报告`:
- NSA、CISA Kubernetes 加固指南 v1.1 的网络安全技术报告。
- CIS Kubernetes 基准 v1.23 的网络安全技术报告。
- Kubernetes pss-baseline，Pod 安全标准
- Kubernetes pss-restricted，Pod 安全标准

### 安装

`trivy-operator-values.yaml`

```
# targetNamespace defines where you want trivy-operator to operate. By
# default, it's a blank string to select all namespaces, but you can specify
# another namespace, or a comma separated list of namespaces.
targetNamespaces: ""

# excludeNamespaces is a comma separated list of namespaces (or glob patterns)
# to be excluded from scanning. Only applicable in the all namespaces install
# mode, i.e. when the targetNamespaces values is a blank string.
excludeNamespaces: ""

# targetWorkloads is a comma seperated list of Kubernetes workload resources
# to be included in the vulnerability and config-audit scans
# if left blank, all workload resources will be scanned
targetWorkloads: "pod,replicaset,replicationcontroller,statefulset,daemonset,cronjob,job"

image:
  repository: "artifactory.thomasroot.local/aquasecurity/trivy-operator"

trivy:
 # repository of the Trivy image
  repository: "artifactory.thomasroot.local/aquasecurity/aquasec/trivy"

  # Registries without SSL. There can be multiple registries with different keys.
  nonSslRegistries: {
    artifactory: "artifactory.thomasroot.local"
  }

  # The registry to which insecure connections are allowed. There can be multiple registries with different keys.
  insecureRegistries: {
    artifactory: "artifactory.thomasroot.local"
  }

# serverCustomHeaders is a comma separated list of custom HTTP headers sent by
  # Trivy client to Trivy server. Only applicable in ClientServer mode.
  #
  # serverCustomHeaders: "foo=bar"

  dbRepository: "artifactory.thomasroot.local/aquasecurity/trivy-db"

  # The Flag to enable insecure connection for downloading trivy-db via proxy (air-gaped env)  
  # Customized because local artifactory repo is "insecure"
  dbRepositoryInsecure: "true"

serviceMonitor:
  # enabled determines whether a serviceMonitor should be deployed
  enabled: true
  namespace: monitoring
  labels:
    release: "prom"   #use your own labels
trivy:
  ignoreUnfixed: true

operator:
  # scanJobTimeout the length of time to wait before giving up on a scan job
  scanJobTimeout: 5m
```

以上`values文件`的包含3块内容

- Environments
- Prometheus serviceMonitor
- Restricting and excluding resources
- Timeouts

`Environments` 部分配置,如果您使用的是自定义镜像注册表，如 Artifactory 或 Harbor，则有必要更新容器镜像和漏洞数据库的路径。

> 请务必注意，在激活此设置之前必须已经安装了 Prometheus

`Prometheus serviceMonitor`配置是给`Prometheus` 做自动发现的,
以及`Prometheus`通过`serviceMonitorSelector`的标签选择器来过滤监控项。

`Restricting and excluding resources`部分配置,当您不想扫描某些名称空间或工作负载时很有用。

`Timeout`部分配置,某些情况下，有必要增加`scanJobTimeout`默认值 `5m`


### Helm Install
```
helm repo add aqua https://aquasecurity.github.io/helm-charts/
helm repo update
```

```
helm upgrade -i trivy-operator aqua/trivy-operator \
--namespace trivy-system \
--create-namespace \
--version 0.12.0 \
--set="trivy.resources.requests.memory=1G" \
--set="trivy.resources.limits.memory=1G" \
--values trivy-operator-values.yaml
```


安装后，Trivy Operator 会扫描集群中的所有图像，并将结果存储在各个pod命名空间中的 k8s的CRD资源中。

```
root@node1:~/trivy-operator# kubectl get crd | grep aqua
clustercompliancereports.aquasecurity.github.io        2023-03-03T08:30:39Z
clusterconfigauditreports.aquasecurity.github.io       2023-03-03T08:30:39Z
clusterinfraassessmentreports.aquasecurity.github.io   2023-03-08T05:38:27Z
clusterrbacassessmentreports.aquasecurity.github.io    2023-03-03T08:30:40Z
configauditreports.aquasecurity.github.io              2023-03-03T08:30:40Z
exposedsecretreports.aquasecurity.github.io            2023-03-03T08:30:40Z
infraassessmentreports.aquasecurity.github.io          2023-03-03T08:30:40Z
rbacassessmentreports.aquasecurity.github.io           2023-03-03T08:30:40Z
vulnerabilityreports.aquasecurity.github.io            2023-03-03T08:30:40Z

```

`clustercompliancereports`：根据CIS 基准和NSA Kubernetes 强化指南创建报告。
`clusterrbacassessmentreports& rbacassessmentreports`：扫描提供有关已安装的不同资源的访问权限的详细信息，并突出显示与ClusterRoles或权限过高的任何问题Roles。
`clusterconfigauditreports`：使用预定义规则或自定义 Open Policy Agent (OPA) 策略对 Kubernetes 资源进行自动配置审计。
`exposedsecretreports`：自动扫描，定位并详细说明集群中暴露的 Secret 的位置。
`infraassessmentreports`：扫描 Kubernetes 核心组件（etcd、apiserver、scheduler、controller-manager 等）设置和配置。
`vulnerabilityreports`：自动扫描容器镜像中的漏洞。

### Grafana指标

`Aqua Trivy` 仪表板的 Grafana ID： `17813`

![-2023-03-09-09-53-134641fb90664835c3.png](https://youjb.com/images/2023/03/09/-2023-03-09-09-53-134641fb90664835c3.png)

![-2023-03-09-10-35-15d37dcf1029bdc2ec.png](https://youjb.com/images/2023/03/09/-2023-03-09-10-35-15d37dcf1029bdc2ec.png)
[Grafana Dashboard for Trivy Operator ](https://gist.github.com/schnatterer/a88a2d35436662cfb7d29dec6058c4d2)
## 参考文档
https://aquasecurity.github.io/trivy-operator/v0.12.0/tutorials/grafana-dashboard/
https://medium.com/datev-techblog/trivy-operator-improve-container-runtime-security-2a1f647dcab
https://community.cloudogu.com/t/continuously-scan-your-kubernetes-cluster-for-security-issues-using-trivy-operator/962
