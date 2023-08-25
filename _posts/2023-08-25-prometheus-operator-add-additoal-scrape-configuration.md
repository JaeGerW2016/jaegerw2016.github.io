---
layout: mypost
title: Prometheus operator additional scrape configuration 配置
categories: [Prometheus]
---

## 背景
在关注到云原生监控体系面临着指标量快速上涨带来的一系列挑战，本文将分享Prometheus oprator模式下中容器监控遇到的问题以及我们的解决和优化方法


## 监控组件负载快速升高
容器化每次部署IP都会变化的特性，导致容器监控指标量相比物理机和虚拟机要高出好几个数量级。同时由于集群规模的不断增加以及业务的快速增长，导致监控 Prometheus、VictoriaMetrics 负载快速增高，给我们容器监控带来极大的挑战。监控总时间序列可以精简为以下的表达式，即与 Pod数量、Pod指标量、指标Label维度数量呈线性关系：

  TotalSeries = PodNum * PerPodMetrics * PerLabelCount

 

而随着集群规模的不断增加以及容器数量的不断增多，监控序列就会快速增长，同时监控组件负载快速升高，会对容器监控系统的稳定性产生影响。

## 解决方案

基于以上的监控项目随着集群规模扩张，监控指标也随之飙升，这就需要我们对一些无用或者优先级低的指标进行优化

首先我们可以到Prometheus的后台查看 `TSDB Status`中 查看`Top 10 series count by metric names`的统计数据

[![photo_2023-08-25_16-38-403bd2500eacd89c15.md.jpeg](https://youjb.com/images/2023/08/25/photo_2023-08-25_16-38-403bd2500eacd89c15.md.jpeg)](https://youjb.com/image/cHE)

这里我们以 `container_threads` 容器线程数 这个Target为例，做`DROP`操作
这个就需要我们将这个查询 kublet组件下面 `/metrics/cAdvisor`下面的Target 配置加到 prometheus的Configuration中

这个Configuration 是Prometheus operator存在 Secret中

```
root@node1:~# kubectl get secret -n monitoring prometheus-prom-kube-prometheus-stack-prometheus
NAME                                               TYPE     DATA   AGE
prometheus-prom-kube-prometheus-stack-prometheus   Opaque   1      170d

```

### 如何导出prometheus配置

```
## export prometheus.yaml
root@node1:~# kubectl get -n monitoring secrets prometheus-prom-kube-prometheus-stack-prometheus -o jsonpath='{.data.prometheus\.yaml\.gz}' | base64 -d | gunzip >prometheus.yaml

## edit prometheus.yaml and generate secret
vi prometheus.yaml
gzip prometheus.yaml
cat prometheus.yaml.gz | base64 -w 0

## replace the secrets 
kubectl edit secrets prometheus-prom-kube-prometheus-stack-prometheus -n monitoring

```

在尝试 编辑这个Secret之后，发现这个操作并不生效，由于这个是被`Prometheus Operator`纳管，无法通过直接修改`prometheus.yaml.gz` 内容生效。

在查阅Prometheus operator的官方Github和官方文档，看到以下一段话

>Note: Starting with v0.39.0, Prometheus Operator requires use of Kubernetes v1.16.x and up.
Deprecation Warning: The custom configuration option of the Prometheus Operator will be deprecated in favor of the additional scrape config option.

## [Addition-scrape-configuration](https://github.com/prometheus-operator/prometheus-operator/blob/main/Documentation/additional-scrape-config.md)

可以看出Prometheus operator后续都把`Custom configuration`都挪到 这个`Addition-scrape-configuration`配置项中

根据文档描述
- 创建 prometheus-additional.yaml
- 生成secrets文件 additional-scrape-configs.yaml
- 应用 secrets
- prometheus 配置重加载


`prometheus-additional.yaml`
```
- job_name: serviceMonitor/monitoring/prom-kube-prometheus-stack-kubelet/3
  honor_labels: true
  kubernetes_sd_configs:
  - role: endpoints
    namespaces:
      names:
      - kube-system
  metrics_path: /metrics/cadvisor
  scheme: https
  tls_config:
    insecure_skip_verify: true
    ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
  bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
  relabel_configs:
  - source_labels:
    - job
    target_label: __tmp_prometheus_job_name
  - action: keep
    source_labels:
    - __meta_kubernetes_service_label_app_kubernetes_io_name
    - __meta_kubernetes_service_labelpresent_app_kubernetes_io_name
    regex: (kubelet);true
  - action: keep
    source_labels:
    - __meta_kubernetes_service_label_k8s_app
    - __meta_kubernetes_service_labelpresent_k8s_app
    regex: (kubelet);true
  - action: keep
    source_labels:
    - __meta_kubernetes_endpoint_port_name
    regex: https-metrics
  - source_labels:
    - __meta_kubernetes_endpoint_address_target_kind
    - __meta_kubernetes_endpoint_address_target_name
    separator: ;
    regex: Node;(.*)
    replacement: ${1}
    target_label: node
  - source_labels:
    - __meta_kubernetes_endpoint_address_target_kind
    - __meta_kubernetes_endpoint_address_target_name
    separator: ;
    regex: Pod;(.*)
    replacement: ${1}
    target_label: pod
  - source_labels:
    - __meta_kubernetes_namespace
    target_label: namespace
  - source_labels:
    - __meta_kubernetes_service_name
    target_label: service
  - source_labels:
    - __meta_kubernetes_pod_name
    target_label: pod
  - source_labels:
    - __meta_kubernetes_pod_container_name
    target_label: container
  - action: drop
    source_labels:
    - __meta_kubernetes_pod_phase
    regex: (Failed|Succeeded)
  - source_labels:
    - __meta_kubernetes_service_name
    target_label: job
    replacement: ${1}
  - source_labels:
    - __meta_kubernetes_service_label_k8s_app
    target_label: job
    regex: (.+)
    replacement: ${1}
  - target_label: endpoint
    replacement: https-metrics
  - source_labels:
    - __metrics_path__
    target_label: metrics_path
    action: replace
  - source_labels:
    - __address__
    target_label: __tmp_hash
    modulus: 1
    action: hashmod
  - source_labels:
    - __tmp_hash
    regex: $(SHARD)
    action: keep
  metric_relabel_configs:
  - source_labels:
    - __name__
    regex: container_threads.*
    action: drop

```
以上内容中 `job_name` 不能重复，否则加载配置时提示报错
>  ts=2023-08-25T01:49:23.086Z caller=main.go:941 level=error msg="Error reloading config" err="couldn't load configuration (--config.file=\"/etc/prometheus/config_out/prometheus.env.yaml\"): parsing YAML file /etc/prometheus/config_out/prometheus.env.yaml: found multiple scrape configs with job name \"serviceMonitor/monitoring/prom-kube-prometheus-stack-kubelet/1\""

### 生成secrets文件

```
kubectl create secret generic additional-scrape-configs --from-file=prometheus-additional.yaml --dry-run=client -oyaml > additional-scrape-configs.yaml

cat additional-configs.yaml
apiVersion: v1
data:
  prometheus-additional.yaml: LSBqb2xxxxxxxxxxx
kind: Secret
metadata:
  creationTimestamp: null
  name: additional-scrape-configs
```

### 应用Secrets

```
kubectl apply -f additional-scrape-configs.yaml -n monitoring
```

### Prometheus 重加载配置文件

```
root@node1:~# kubectl get svc -n monitoring prometheus-operated
NAME                  TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE
prometheus-operated   ClusterIP   None         <none>        9090/TCP   170d


root@node1:~# kubectl port-forward -n monitoring svc/prometheus-operated 9090:9090
Forwarding from 127.0.0.1:9090 -> 9090

## Prometheus enable with --web.enable-lifecycle option

root@node1:~# curl -X POST http://localhost:9090/-/reload

root@node1:~# kubectl logs prometheus-prom-kube-prometheus-stack-prometheus-0 -n monitoring -f

ts=2023-08-25T09:22:37.643Z caller=main.go:1234 level=info msg="Completed loading of configuration file" filename=/etc/prometheus/config_out/prometheus.env.yaml totalDuration=210.847389ms db_storage=1.411µs remote_storage=2.248µs web_handler=913ns query_engine=1.041µs scrape=2.084842ms scrape_sd=5.917352ms notify=15.4µs notify_sd=375.219µs rules=198.11146ms tracing=6.717µs
```
查看Prometheus的配置文件

[![QQ20230825172615fb0b254d277d839c.md.png](https://youjb.com/images/2023/08/25/QQ20230825172615fb0b254d277d839c.md.png)](https://youjb.com/image/cHz)

以上additional-scrape-config才算添加成功，也对Promethus的有了新的认识，以此文记录。