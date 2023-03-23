---
title:  Kubernetes笔记 ResourceQuotas和LimitRange
date: 2019-08-06 17:13
tag: kubernetes
---

当多个用户或团队共享具有固定数目节点的集群时，人们会担心有人使用的资源超出应有的份额。

`ResourceQuotas`是帮助管理员解决这一问题的工具。

资源配额， 通过 `ResourceQuota` 对象来定义， 对每个namespace的资源消耗总量提供限制。 它可以按类型限制`namespace`下可以创建的对象的数量，也可以限制可被该项目以资源形式消耗的计算资源的总量。
- 用户在namespace下创建资源 (pods、 services等)，同时配额系统会跟踪使用情况，来确保其不超过 资源配额中定义的硬性资源限额。
- 如果资源的创建或更新违反了配额约束，则请求会失败，并返回 HTTP状态码 403 FORBIDDEN ，以及说明违反配额 约束的信息。
- 如果namespace下的计算资源 （如 cpu 和 memory）的配额被启用，则用户必须为这些资源设定请求值（request） 和约束值（limit），否则配额系统将拒绝Pod的创建。

> 提示: 可配合 LimitRange 准入控制器来为没有设置计算资源需求的Pod设置`request`和`limits`默认值。



### 启用[ResourceQuotas](https://kubernetes.io/zh/docs/concepts/policy/resource-quotas/)

当 apiserver 的 `--admission-control=` 参数中包含 `ResourceQuota` 时，资源配额会被启用。

当namespace中存在一个 `ResourceQuota` 对象时，该namespace即开始实施资源配额管理。 一个namespace中最多只应存在一个 `ResourceQuota` 对象

>  Kubernetes 的版本在1.10+ 之后 启用的的参数`--enable-admission-plugins= `，`--admission-control=` 将在后续被废弃

 

资源配额分为以下三种：

- 计算资源配额：`cpu`，`memory`
- 存储资源配置：requests.storage的容量，pvc数量，某storage class下的requests.storage的容量和pvc数量限制
- 对象数量配置：`pods`,`configmap`，`service`,`services.loadbalancers`,`services.nodeports`数量。

> 用户可能希望在namespace中为pod设置配额,来避免有用户创建过多的pod，从而耗尽集群提供的pod IP地址

#### Kubectl 支持创建、更新和查看配额

```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: myspace
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-resources
spec:
  hard:
    pods: "10"
    requests.cpu: "1"
    requests.memory: 1Gi
    limits.cpu: "2"
    limits.memory: 2Gi
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: object-counts
spec:
  hard:
    configmaps: "10"
    persistentvolumeclaims: "4"
    replicationcontrollers: "20"
    secrets: "10"
    services: "10"
    services.loadbalancers: "2"
```

> 如果配额中指定了 `requests.cpu` 或 `requests.memory` 的值，那么它要求每个进来的容器针对这些资源有明确的请求。 如果配额中指定了 `limits.cpu` 或 `limits.memory`的值，那么它要求每个进来的容器针对这些资源指定明确的约束

针对这个`myspace`资源配额限制：

- 最大`pods`数量为10

- CPU需求总量不能超过1
- CPU限额总量不能超过2
- 内存需求总量不能超过1Gi
- 内存限额总量不能超过2Gi

- 允许存在的service的数量最大为10
- 允许存在的configmap的数量最大为10
- 允许存在的pvc的数量最大为4

- 允许存在的replication controllers的数量最大为20
- 允许存在的secret的数量最大为10
- 允许存在的load balancer类型的service的数量最大为2

### [LimitRanges](https://kubernetes.io/docs/concepts/policy/limit-range/)

`Resource Quota`区分的粒度是`namespace`，其目的是为了不同namespace之间的公平，防止某些流氓team占用了太多的资源。而`limitRange`区分的粒度则是`container`，则是在为了在同一个namespace下，限制container的最大最小值。

> 在设置了resourceQuota的namespace下，如果用户创建Pod时没有指定limit/request，默认是无法创建的（403 FORBIDDEN，此时可以通过limitRanges来配置该namespace下容器*默认*的limit/request。

### 启用LimitRanges

默认情况下启用限制范围支持。当apiserver `--enable-admission-plugins=`标志具有`LimitRanger`允许控制器作为其参数之一时，它被启用

#### 使用限制范围创建的策略示例如下

`limit-mem-cpu-container.yaml`

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: limit-mem-cpu-per-container
  namespace: myspace
spec:
  limits:
  - max:
      cpu: "1"
      memory: "1Gi"
    min:
      cpu: "100m"
      memory: "100Mi"
    default:
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:
      cpu: "200m"
      memory: "256Mi"
    type: Container
```

- `default` 即该namespace配置resourceQuota时，创建container的默认limit上限

- `defaultRequest`：即该namespace配置resourceQuota时，创建container的默认request上限
- `max`：即该namespace下创建container的资源最大值
- `min`：即该namespace下创建container的资源最小值

> 其中： min <= defaultRequest <= default <= max

```sh
root@k8s-master-1:~/k8s_manifests/quota-resource# kubectl describe limitranges/limit-mem-cpu-per-container -n myspace
Name:       limit-mem-cpu-per-container
Namespace:  myspace
Type        Resource  Min    Max  Default Request  Default Limit  Max Limit/Request Ratio
----        --------  ---    ---  ---------------  -------------  -----------------------
Container   cpu       100m   1    200m             500m           -
Container   memory    100Mi  1Gi  256Mi            512Mi          -
```

`limit-range-pod-1.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: busybox1
spec:
  containers:
  - name: busybox-cnt01
    image: busybox
    command: ["/bin/sh"]
    args: ["-c", "while true; do echo hello from cnt01; sleep 10;done"]
    resources:
      requests:
        memory: "100Mi"
        cpu: "100m"
      limits:
        memory: "200Mi"
        cpu: "500m"
  - name: busybox-cnt02
    image: busybox
    command: ["/bin/sh"]
    args: ["-c", "while true; do echo hello from cnt02; sleep 10;done"]
    resources:
      requests:
        memory: "100Mi"
        cpu: "100m"
  - name: busybox-cnt03
    image: busybox
    command: ["/bin/sh"]
    args: ["-c", "while true; do echo hello from cnt03; sleep 10;done"]
    resources:
      limits:
        memory: "200Mi"
        cpu: "500m"
  - name: busybox-cnt04
    image: busybox
    command: ["/bin/sh"]
    args: ["-c", "while true; do echo hello from cnt04; sleep 10;done"]
```

创建`busybox1`Pod：

```shell
root@k8s-master-1:~/k8s_manifests/quota-resource# kubectl apply -f limit-range-pod-1.yaml -n myspace
```

查看`busybox-cnt01` 资源配置

```shell
root@k8s-master-1:~/k8s_manifests/quota-resource# kubectl get  po/busybox1 -n myspace -o json | jq ".spec.containers[0].resources"
{
  "limits": {
    "cpu": "500m",
    "memory": "200Mi"
  },
  "requests": {
    "cpu": "100m",
    "memory": "100Mi"
  }
}
```

- Pod中的`busybox-cnt01`Container `busybox`定义`requests.cpu=100m`和`requests.memory=100Mi`。
- `100m <= 500m <= 1` ，容器cpu限制（500m）落在授权的CPU限制范围内。
- `100Mi <= 200Mi <= 1Gi` ，容器内存限制（200Mi）落在授权的内存限制范围内。
- 没有CPU /内存的请求/限制比率验证，因此容器有效并创建

查看`busybox-cnt02`资源配置

```shell
root@k8s-master-1:~/k8s_manifests/quota-resource# kubectl get  po/busybox1 -n myspace -o json | jq ".spec.containers[1].resources"
{
  "limits": {
    "cpu": "500m",
    "memory": "512Mi"
  },
  "requests": {
    "cpu": "100m",
    "memory": "100Mi"
  }
}
```

- Pod中的`busybox-cnt02`Container `busybox`定义`requests.cpu=100m`和`requests.memory=100Mi`但没定义CPU和内存的`limits`

- 容器没有`limits`部分，limit-mem-cpu-per-container LimitRange对象中定义的默认限制将注入此容器`limits.cpu=500mi和`limits.memory=512Mi`。

- `100m <= 500m <=1` ，容器cpu限制（500m）落在授权的CPU限制范围内。

- `100Mi <= 512Mi <= 1Gi` ，容器内存限制（512Mi）落在授权的内存限制范围内。

  > 如果container只设置了request，没设置limit，则最终container的limit为default设置的值。

查看`busybox-cnt03`资源配置

```shell
root@k8s-master-1:~/k8s_manifests/quota-resource# kubectl get  po/busybox1 -n myspace -o json | jq ".spec.containers[2].resources"
{
  "limits": {
    "cpu": "500m",
    "memory": "200Mi"
  },
  "requests": {
    "cpu": "500m",
    "memory": "200Mi"
  }
}
```

- Pod中的`busybox-cnt03`Container `busybox`定义`limits.cpu=500m`和`limits.memory=200Mi`，但没有`requests`对CPU和内存
- 容器没有定义`requests`部分，limit-mem-cpu-per-container LimitRange中定义的defaultRequest不用于填充其限制部分，但容器定义的限制设置为请求`limits.cpu=500m`和`limits.memory=200Mi`。
- `100m <= 500m <= 1` ，容器cpu限制（500m）落在授权的CPU限制范围内。
- `100Mi <= 200Mi <= 1Gi` ，容器内存限制（200Mi）落在授权的内存限制范围内。
- 没有设置请求/限制比率，因此容器有效并且已创建

> 注意：如果container的resources部分设置了limit，没有设置request，那么最终创建出来的container，其request=limit。而不是想当然的是defaultRequest。

查看`busybox-cnt04`资源配置

```
root@k8s-master-1:~/k8s_manifests/quota-resource# kubectl get  po/busybox1 -n myspace -o json | jq ".spec.containers[3].resources"
{
  "limits": {
    "cpu": "500m",
    "memory": "512Mi"
  },
  "requests": {
    "cpu": "200m",
    "memory": "256Mi"
  }
}
```

- Pod中的`busybox-cnt03`Container `busybox`定义中没有`requests`和`limits`
- 容器没有定义`limits`部分，limit-mem-cpu-per-container LimitRange中定义的默认限制用于填充其请求 `limits.cpu=500m and` `limits.memory=512Mi`。
- 容器没有定义`requests`部分，limit-mem-cpu-per-container LimitRange中定义的defaultRequest用于填充请求部分`requests.cpu = 200m`和`requests.memory = 256Mi`
- `100m <=500m <= 1` ，容器cpu限制（500m）落在授权的CPU限制范围内。
- `100Mi <= 512Mi <= 1Gi` ，容器内存限制（512Mi）属于授权的内存限制范围。



`requests`和`limits`的设置会影响Pod的[**Qos服务等级**](https://k8smeetup.github.io/docs/tasks/configure-pod-container/quality-service-pod/)，如果系统中的所有节点都没有剩余资源来填充请求，则Pod将进入`pending`状态。由于CPU可以被压缩，Kubernetes将减少该容器的调度时间，并不会直接杀死Pod。内存无法压缩，因此如果Node耗尽内存，Kubernetes需要开始根据Pod的**Qos服务等级**和**Pod的优先级**，进行驱逐Pod或者杀死Pod,并首先终止最低优先级的pod。如果所有Pod具有相同的优先级，Kubernetes将终止最多其请求的Pod。。

### 参考文档

https://cloud.google.com/blog/products/gcp/kubernetes-best-practices-resource-requests-and-limits

https://kubernetes.io/zh/docs/concepts/policy/resource-quotas/

https://kubernetes.io/docs/concepts/policy/limit-range/

https://k8smeetup.github.io/docs/tasks/configure-pod-container/quality-service-pod/
