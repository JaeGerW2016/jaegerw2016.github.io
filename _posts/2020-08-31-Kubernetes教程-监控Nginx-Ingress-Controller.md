---
layout:     post
title:      Kubernetes教程：如何监控Nginx Ingress Controller的Metrics指标
subtitle:   配置集群现有Nginx Ingress Controller及Prometheus operator
date:       2020-08-31 
author:     J
catalog: true
tags:
    - kubernetes
    - Prometheus
---

`Ingress` 是从`Kubernetes`集群外部访问集群内部服务的入口。你可以在`Ingress`配置中提供外部可访问的URL、负载均衡、SSL、基于名称的虚拟主机等。

这次我们主要以`Nginx Ingress`为例子，来介绍如何将现有的的`Nginx Ingress Controller`的`Metrics`指标添加到`Prometheus`中以及以`Grafana Dashboard`的形式展现。

`Nginx Ingress` 是 `Kubernetes Ingress` 的一种实现，它通过` watch Kubernetes` 集群的` Ingress ``资源，将 `Ingress` 规则转换成 `Nginx `的配置，然后让` Nginx` 来进行 `7 `层的流量转发:

<img src="https://i.loli.net/2020/08/31/OicVEPl6Iv3qmsg.jpg" alt="nginx-ingress-on-tke-1.jpg" style="zoom:67%;" />

### deployment

针对现有的`Nginx Ingress Controller`的deployment 需要添加2个新的`annotations`

- `prometheus.io/scrape: "true"`  #启用Prometheus的自动发现
- `prometheus.io/port: "10254"`  #指定Metrics的端口

暴露Metrics端口10254

```
...
        - containerPort: 10254
          name: metrics
          protocol: TCP
...
```

### Service

```yaml
...
  - name: metrics
    port: 10254
    protocol: TCP
...
```

### ServiceMonitor

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: nginx-ingress-controller-metrics
  namespace: <Your Namespce> #根据自己Prometheus的namespace来修改
  labels:
    app: nginx-ingress
    release: prom #根据自己的Prometheus ServiceMonitorSelector的标签修改
spec:
  endpoints:
  - interval: 30s
    port: metrics
  selector:
    matchLabels:
      app: nginx-ingress
      release: nginx-ingress
  namespaceSelector:
    matchNames:
    - kube-system #根据nginx ingress controller的namespace来匹配
```

### Grafana

在`Grafana`的管理界面导入官方Dashboard ID `9614` 

[*NGINX Ingress controller*](https://grafana.com/grafana/dashboards/9614)

### 结果

```yaml
# kubectl get ep -l app=nginx-ingress -l component=controller
NAME                       ENDPOINTS                                             AGE
nginx-ingress-controller   192.168.2.12:80,192.168.2.12:10254,192.168.2.12:443   133d
```

![nginx-ingress-controller-target.png](https://i.loli.net/2020/08/31/OBLnfQWdM2uxUP7.png)

![nginx-ingress-controller-Dashboard.png](https://i.loli.net/2020/08/31/Hf3kFhPQEMU9GWp.png)
