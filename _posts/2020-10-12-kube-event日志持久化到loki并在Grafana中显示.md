---
layout:     post
title:     kubernetes event事件持久化到loki并在Grafana中展示
subtitle: 	LokiEventCollector组件如何将kube event集群日志持久化
date:       2020-10-12
author:     J
catalog: true
tags:
    - kubernetes
    - loki
---

`Kubernetes` `Event`是`Kubernetes`中的一种资源类型，当其他资源具有状态更改，错误或其他应广播到系统的消息时，会自动创建该Event。尽管没有太多有关`Kubernetes` `Event`的文档，但在调试`Kubernetes`集群中的问题时，它们是宝贵的资源。由于`Kubernetes` Event在设计的时候默认只保留几个小时内的资源信息，所以对它的数据持久化是非常有必要的操作。

在本文中，我们将学习如何查看`Kubernetes` `Event`，了解一些特定的`Kubernetes Event`类型以及讨论如何监视`Kubernetes Event `、最后做一个数据持久化到Loki日志系统并在`Grafana`中做一个简单的展示。

### 如何获取Event事件

如果您已经使用`Kubernetes`已有一段时间了，那么您很可能以前曾经看过`Kubernetes Event`事件。使用`kubectl`，当您描述无法正常工作的容器或其他资源时，您可能已经看到它们。

在大多数情况下，当您尝试调试特定资源的问题时很容易看到Event事件。`kubectl describe pod <podname>`例如，使用将在pod的输出末尾显示Event事件。仅会显示最近发生的事件（几个小时之内）。 

![20201012153242.jpg](https://i.loli.net/2020/10/12/DWYwXZF9CVnqKE8.jpg)

查看事件的另一种方法是直接从资源API获取事件。可以使用来完成此操作`kubectl get events`，它显示系统中所有资源的最近事件。

![20201012153422.jpg](https://i.loli.net/2020/10/12/Ekv6JunPgzZQHdW.jpg)

在实际生产环境中，我们并不需要显示全部的Event事件，这个时候就需要我们对这些事件做一些过滤和筛选。

使用`--field-selector`调用的参数来过滤事件很容易解决。以下是一些快速的示例，您会发现它们很有用：

**仅Warning类型的事件**

```shell
kubectl get events --field-selector type=Warning
```

**非Pod类型的事件**

```shell
kubectl get events --field-selector involvedObject.kind!=Pod
```

**来自name=node1的事件**

```shell
kubectl get events --field-selector involvedObject.kind=Node,involvedObject.name=node1
```

**非Normal类型事件**

```shell
kubectl get events --field-selector type!=Normal
```

### 如何在集群内获取Event事件

#### 1. 自定义 `EventController`

通过在集群部署组件`LokiEventCollector` 该组件内包含一个`EventController` 然后通过`client-go`库中的Informers机制来监听apiserver，从apiserver中接受该类型资源的变化，由用户注册的回调函数对资源变化进行处理，将Event事件通过Loki的api推送到Loki中做存储。

#### 2. `leader election`选举

同时基于高可用的考虑，在参考了`kubernetes`的j控制组件中的 `kube-scheduler`和`kube-controller-manager`，它们虽然在三个 master 中均部署了三份副本，但是实际上只有一个在真正工作。三个组件如何在集群中相互协调决定哪一个真正工作，其背后原理是 `leader election`。

多个成员参与 leader election 的行为中，成为分为两类：

- leader
- candidate

其中 leader 为真正工作的成员，其他成员为 candidate ，它们并没有在执行工作而是随时等待成为 leader 并开始工作。在它们运行之初都是 candidate，初始化时都会去获取一个唯一的锁，谁抢到谁就成为 leader 并开始工作，在 `client-go` 中这个锁就是 apiserver 中的一个资源，资源类型一般是 `CoordinationV1()` 资源组中的 `Lease` 资源。

在 leader 被选举成功之后，leader 为了保住自己的位置，需要定时去更新这个 `Lease` 资源的状态，即一个时间戳信息，表明自己有在一直工作没有出现故障，这一操作称为续约。其他 candidate 也不是完全闲着，而是也会定期尝试获取这个资源，检查资源的信息，时间戳有没有太久没更新，否则认为原来的 leader 故障失联无法正常工作，并更新此资源的 holder 为自己，成为 leader 开始工作并同时定期续约。

#### 3. golang的`reflect反射机制`来对Event `json`嵌套格式内容进行重组

通过`golang`的`reflect反射机制`来对Event `json`嵌套格式的内容，进行分类清洗重组

`example-event.json`

```json
        {
            "apiVersion": "v1",
            "count": 61,
            "eventTime": null,
            "firstTimestamp": "2020-09-23T07:22:15Z",
            "involvedObject": {
                "apiVersion": "v1",
                "fieldPath": "spec.containers{kubernetes-dashboard}",
                "kind": "Pod",
                "name": "kubernetes-dashboard-556b9ff8f8-gf6lg",
                "namespace": "kube-system",
                "resourceVersion": "39702943",
                "uid": "4f22e027-6a03-4ec1-8ce9-83850dd3d431"
            },
            "kind": "Event",
            "lastTimestamp": "2020-10-12T08:51:05Z",
            "message": "Liveness probe failed: Get https://172.20.1.96:8443/: net/http: request canceled (Client.Timeout exceeded while awaiting headers)",
            "metadata": {
                "creationTimestamp": "2020-10-12T08:51:05Z",
                "name": "kubernetes-dashboard-556b9ff8f8-gf6lg.163758b70d2070c1",
                "namespace": "kube-system",
                "resourceVersion": "45897350",
                "selfLink": "/api/v1/namespaces/kube-system/events/kubernetes-dashboard-556b9ff8f8-gf6lg.163758b70d2070c1",
                "uid": "270a4ca9-e24c-4a40-b7e2-784352fd51ac"
            },
            "reason": "Unhealthy",
            "reportingComponent": "",
            "reportingInstance": "",
            "source": {
                "component": "kubelet",
                "host": "node2"
            },
            "type": "Warning"
        }
```

`https://github.com/JaeGerW2016/LokiEventCollector/blob/master/receiver/loki.go`

```go
...
func formatEvent(e v1.Event) string {
	var b strings.Builder
	t := reflect.TypeOf(e)
	v := reflect.ValueOf(e)
	for i := 0; i < v.NumField(); i++ {
		if v.Field(i).CanInterface() {
			if v.Field(i).Type().Kind() == reflect.Struct {
				structField := v.Field(i).Type()
				for j := 0; j < structField.NumField(); j++ {
					//filter string slice map struct which value is empty
					if interfaceValueAssert(v.Field(i).Field(j).Interface()) {
						continue
					}
					b.WriteString(fmt.Sprintf("%s:%v, ", structField.Field(j).Name, v.Field(i).Field(j).Interface()))
				}
				continue
			}
			if t.Field(i).Name == "Message" {
				m := trimQuotes(fmt.Sprintf("%v", v.Field(i).Interface()))
				b.WriteString(fmt.Sprintf("%s:%v, ", t.Field(i).Name, m))
				continue
			}
			if interfaceValueAssert(v.Field(i).Interface()) {
				continue
			}
			b.WriteString(fmt.Sprintf("%s:%v, ", t.Field(i).Name, v.Field(i).Interface()))
		}

	}
	return b.String()
}
...
```

### 部署`LokiEventCollector`、`Loki`、`Grafana`

**部署`LokiEventCollector`**

```shell
kubectl apply -f https://raw.githubusercontent.com/JaeGerW2016/LokiEventCollector/master/deploy/loki-event-collector.yaml
```

**部署`Loki`**

```shell
helm repo add loki https://grafana.github.io/loki/charts
helm repo update
helm upgrade --install loki loki/loki -n <YOUR-NAMESPACE>
```

**部署`Grafana`**

```shell
helm install loki-grafana stable/grafana -n <YOUR-NAMESPACE>
```

Loki 只会对你的日志元数据标签（就像 Prometheus 的标签一样）进行索引，而不会对原始的日志数据进行全文索引。然后日志数据本身会被压缩，并以 chunks（块）的形式存储在对象存储（比如 S3 或者 GCS）甚至本地文件系统。一个小的索引和高度压缩的 chunks 可以大大简化操作和降低 Loki 的使用成本。

可以通过Log Labels来做分组，LokiEventCollector默认4个分组`_host`  `_namespace` `_type` `_kind`对Event事件分组

### 验证

`kubectl CLI`显示Pod运行情况

![20201012172804.jpg](https://i.loli.net/2020/10/12/VWAT2iZvNSjHxGp.jpg)

`Grafana界面显示`

![20201012173323.jpg](https://i.loli.net/2020/10/12/Q2Pe6fKgZ5R4qDr.jpg)



### 参考文档

https://github.com/JaeGerW2016/LokiEventCollector

https://www.bluematador.com/blog/kubernetes-events-explained