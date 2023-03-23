---
layout:     post
title:      Kubernetes NetworkPolicy 探秘
subtitle:   了解Kubernete复杂网络策略
date:       2020-05-07
author:     J
catalog: true
tags:
    - kubernetes
---



### 网络策略



 网络策略（NetworkPolicy）是一种关于pod间及pod与其他网络端点间所允许的通信规则的规范。`NetworkPolicy` 资源使用标签选择pod，并定义选定pod所允许的通信规则。

默认情况下，Pod之间是非隔离的，它们接受任何来源的流量，并且网络策略的生效是需要CNI插件的支持：

常见的支持 NetworkPolicy 的网络插件有：

- Calico
- Cilium
- Kube-Router
- Romana
- Weave Net

### NetworkPolicy的字段定义

```shel
root@node1:~# kubectl explain networkpolicies
KIND:     NetworkPolicy
VERSION:  networking.k8s.io/v1

DESCRIPTION:
     NetworkPolicy describes what network traffic is allowed for a set of Pods

FIELDS:
   apiVersion	<string>
     APIVersion defines the versioned schema of this representation of an
     object. Servers should convert recognized schemas to the latest internal
     value, and may reject unrecognized values. More info:
     https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#resources

   kind	<string>
     Kind is a string value representing the REST resource this object
     represents. Servers may infer this from the endpoint the client submits
     requests to. Cannot be updated. In CamelCase. More info:
     https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#types-kinds

   metadata	<Object>
     Standard object's metadata. More info:
     https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#metadata

   spec	<Object>
     Specification of the desired behavior for this NetworkPolicy.
```

其中`apiVersion` `kind` `metadata` 属于**必填**字段，`spec`字段中包含了一个`metadata`中的`namespace`中定义特定网络策略所需的所有信息。

`NetworkPolicy` 这个资源属于命名空间级别的，因此`metadata `中的 `namespace` 不可省略，否则只会对`default `下的满足条件的 Pod 生效。

### 重点来解析下`spec`字段中的一些缺省值和含义

```shell
root@node1:~# kubectl explain networkpolicies.spec --recursive
KIND:     NetworkPolicy
VERSION:  networking.k8s.io/v1

RESOURCE: spec <Object>

DESCRIPTION:
     Specification of the desired behavior for this NetworkPolicy.

     NetworkPolicySpec provides the specification of a NetworkPolicy

FIELDS:
   egress	<[]Object>
      ports	<[]Object>
         port	<string>
         protocol	<string>
      to	<[]Object>
         ipBlock	<Object>
            cidr	<string>
            except	<[]string>
         namespaceSelector	<Object>
            matchExpressions	<[]Object>
               key	<string>
               operator	<string>
               values	<[]string>
            matchLabels	<map[string]string>
         podSelector	<Object>
            matchExpressions	<[]Object>
               key	<string>
               operator	<string>
               values	<[]string>
            matchLabels	<map[string]string>
   ingress	<[]Object>
      from	<[]Object>
         ipBlock	<Object>
            cidr	<string>
            except	<[]string>
         namespaceSelector	<Object>
            matchExpressions	<[]Object>
               key	<string>
               operator	<string>
               values	<[]string>
            matchLabels	<map[string]string>
         podSelector	<Object>
            matchExpressions	<[]Object>
               key	<string>
               operator	<string>
               values	<[]string>
            matchLabels	<map[string]string>
      ports	<[]Object>
         port	<string>
         protocol	<string>
   podSelector	<Object>
      matchExpressions	<[]Object>
         key	<string>
         operator	<string>
         values	<[]string>
      matchLabels	<map[string]string>
   policyTypes	<[]string>

```

#### 生效主体：定义我们希望此规则应用特定标签的Pod

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: test-network-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      role: db
```

该规则指定了命名空间为 default， 选择了其中所有包含 `role=db` 这个 label 的 Pod

#### 规则类型：定义我们将使用的策略类型：

```yaml
policyTypes:
  - Ingress
  - Egress
```

定义了入站流量规则与出站流量规则

#### 定义具体Ingress规则：

```yaml
ingress:
  - from:
    - ipBlock:
        cidr: 172.17.0.0/16
        except:
        - 172.17.1.0/24
    - namespaceSelector:
        matchLabels:
          project: myproject
    - podSelector:
        matchLabels:
          role: frontend
    ports:
    - protocol: TCP
      port: 6379
```

对于入站流量,分3个维度来做策略，放行源地址来自 cidr 172.17.0.0/16 除了 172.17.1.0/24 之外的流量，放行来自有`project=myproject` 这个label的namespace中的流量，放行 default 命名空间下有 label `role=frontend` 的 Pod 的流量，并限定这些流量只能访问到 `role=db` 这个 label 的 Pod 的 TCP 6379端口。

#### 定义具体Engress规则

```yaml
  egress:
  - to:
    - ipBlock:
        cidr: 10.0.0.0/24
    ports:
    - protocol: TCP
      port: 5978
```

对于出站流量，分一个维度来做策略，只放行其访问目的地址属于 cidr 10.0.0.0/24 中，且端口为 TCP 5978的流量。

### 完整示例策略

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: test-network-policy
  namespace: default 
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - ipBlock:
        cidr: 172.17.0.0/16
        except:
        - 172.17.1.0/24
    - namespaceSelector:
        matchLabels:
          project: myproject
    - podSelector:
        matchLabels:
          role: frontend
    ports:
    - protocol: TCP
      port: 6379
  egress:
  - to:
    - ipBlock:
        cidr: 10.0.0.0/24
    ports:
    - protocol: TCP
      port: 5978
```

> 需要注意的是 NetworkPolicy 选中的 Pod 只能是与 NetworkPolicy 同处一个 namespace 中的 Pod,因此对于有些规则可能需要在多个命名空间中分别设置。或者使用非原生的网络策略定义

参考文档：

https://www.kubernetes.org.cn/5439.html

https://medium.com/swlh/k8s-network-policies-demystified-and-simplified-18f5ea9848e2

https://github.com/ahmetb/kubernetes-network-policy-recipes