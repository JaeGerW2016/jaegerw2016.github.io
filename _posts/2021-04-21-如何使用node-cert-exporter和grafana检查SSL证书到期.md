---
layout:     post
title:     如何通过Node-cert-exporter和Prometheus以及Grafana监控SSL证书有效性
subtitle: 	保证Kubernetes环境证书的有效性
date:       2021-04-23
author:     J
catalog: true
tags:
    - Prometheus
    - Grafana
---

日常我们对于`Kubernetes`环境下的系统及业务的监控会比较关注，有主机类针对宿主机Node的运行状态、有业务类针对应用性能的`APM`链路追踪等，往往很容易忽略证书类的有效期限的监控，这篇博客的内容就是针对随着时间的流逝，这些`SSL`证书很有可能会过期，从而影响`Kubernetes`系统的日常加密通信。

我们将采用一种简单的Prometheus和node-cert-exporter检查`SSL`证书的到期日期，并在`Grafana`中可视化这些日期数据，并配置相应的告警信息提醒，做到提前更新这些过期的`SSL`证书，保障系统及业务的正常运行。

### 前提

1. `Kubernetes v1.16+`

2. `Prometheus`

3. `Grafana`

4. `AlertManager`

   以上组件准备就绪

### Node-cert-exporter

以`DaemonSet`的形式在`Kubernetes`集群环境运行, `node-cert-exporter`的`serviceType`为`ClusterIP`

```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: monitoring
spec:
  finalizers:
  - kubernetes
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: node-cert-exporter
  namespace: monitoring
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  labels:
    app: node-cert-exporter
  name: node-cert-exporter
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: node-cert-exporter
  template:
    metadata:
      name: node-cert-exporter
      labels:
        app: node-cert-exporter
      annotations:
        prometheus.io/scrape: 'true'
        prometheus.io/port: '9117'
    spec:
      containers:
      - image: amimof/node-cert-exporter:latest
        args:
        - "--v=2"
        - "--logtostderr=true"
        - "--path=/host/etc/ssl,/host/etc/kubernetes/pki/,/host/etc/kubernetes/ssl"
        env:
          - name: NODE_NAME
            valueFrom:
              fieldRef:
                fieldPath: spec.nodeName
        name: node-cert-exporter
        ports:
        - containerPort: 9117
          name: http
          protocol: TCP
        resources:
          limits:
            cpu: 100m
            memory: 128Mi
          requests:
            cpu: 100m
            memory: 128Mi
        volumeMounts:
        - mountPath: /host/etc
          name: etc
          readOnly: true
      serviceAccount: node-cert-exporter
      serviceAccountName: node-cert-exporter
      volumes:
      - hostPath:
          path: /etc
          type: ""
        name: etc
  updateStrategy:
    type: RollingUpdate
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: node-cert-exporter
  name: node-cert-exporter
  namespace: monitoring
spec:
  ports:
  - name: http
    port: 9117
    protocol: TCP
    targetPort: 9117
  selector:
    app: node-cert-exporter
  type: ClusterIP
```



```shell
root@node1:~# kubectl get daemonset,svc,pods -n monitoring -l app=node-cert-exporter
NAME                                DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
daemonset.apps/node-cert-exporter   4         4         4       4            4           <none>          4h34m

NAME                         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/node-cert-exporter   ClusterIP   10.244.35.128   <none>        9117/TCP   5h36m

NAME                           READY   STATUS    RESTARTS   AGE
pod/node-cert-exporter-lspw7   1/1     Running   0          4h34m
pod/node-cert-exporter-pr62m   1/1     Running   0          4h34m
pod/node-cert-exporter-qrbs5   1/1     Running   0          4h34m
pod/node-cert-exporter-tnnrc   1/1     Running   0          4h34m
```

### Prometheus 

自定义`serviceMonitor`文件

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  labels:
    app: node-cert-exporter
    release: prom
  name: node-cert-exporter
  namespace: monitoring
spec:
  endpoints:
  - path: /metrics
    port: http
  namespaceSelector:
    matchNames:
    - monitoring
  selector:
    matchLabels:
      app: node-cert-exporter
```

> 这边有一个注意事项，需要在`.metadata,labels`注释一个`release:prom` 这个是根据Prometheus的`serviceMonitorSelector`选择器的标签匹配有关

```yaml
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
... 省略部分
  serviceMonitorNamespaceSelector: {}
  serviceMonitorSelector:
    matchLabels:
      release: prom
...
```

![](https://ftp.bmp.ovh/imgs/2021/04/5af7f64e47a75592.jpg)

说明`node-cert-exporter`的`/metrics`接口能正常被`Prometheus`的抓取到，`Prometheus`这边的服务发现正常

### 创建`Grafana`仪表盘

#### 导入`Grafana`仪表板

![](https://i.bmp.ovh/imgs/2021/04/126a14de0bc6d926.png)

#### 查找`grafana Dashboard Id`

![](https://i.bmp.ovh/imgs/2021/04/4cf454157e1297f8.png)

#### 导入Dashboard ID为9999的仪表盘

![](https://ftp.bmp.ovh/imgs/2021/04/1271f852246efe55.jpg)

#### 最终展示的`Dashboard`仪表盘

![](https://ftp.bmp.ovh/imgs/2021/04/981e9fef7d90227f.jpg)

#### SSL证书到期警报

添加`PrometheusRule` 告警规则`node-cert-exporter-prometheusrule.yaml`

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  labels:
    app: prometheus-operator
    chart: prometheus-operator-8.15.11
    heritage: Helm
    release: prom
  name: prom-prometheus-operator-node-cert-exporter
  namespace: monitoring
spec:
  groups:
  - name: node-cert-exporter
    rules:
    - alert: SSLCertExpiringSoon
      expr: sum(ssl_certificate_expiry_seconds{}) by (instance, path) < 86400 * 30
      for: 1h
      labels:
        severity: warning
```

设置Alert告警消息机制 （这边通过slack 消息发送）

```yaml
global:
  resolve_timeout: 5m
receivers:
- name: "null"
- name: slack_webhook
  slack_configs:
  - api_url: https://hooks.slack.com/services/XXXXXX/XXXXXXXXXXXXXXXXXX
    channel: alertmanager
    color: '{{ if eq .Status "firing" }}danger{{ else }}good{{ end }}'
    fallback: '{{ template "slack.default.fallback" . }}'
    icon_emoji: '{{ template "slack.default.iconemoji" . }}'
    icon_url: '{{ template "slack.default.iconurl" . }}'
    pretext: '{{ .CommonAnnotations.summary }}'
    send_resolved: false
    text: |-
      {{ range .Alerts }}
        *Alert:* `{{ .Labels.alertname }}` - `{{ .Labels.severity }}`
        *Details:*
        {{ range .Labels.SortedPairs }} • *{{ .Name }}:* `{{ .Value }}`
        {{ end }}
      {{ end }}
    title: '{{ template "slack.default.title" . }}'
    title_link: '{{ template "slack.default.titlelink" . }}'
    username: '{{ template "slack.default.username" . }}'
route:
  group_by:
  - alertname
  - cluster
  - service
  group_interval: 5m
  group_wait: 30s
  receiver: slack_webhook
  repeat_interval: 15m
  routes:
  - match:
      alertname: Watchdog
    receiver: "null"
  - match:
      alertname: CPUThrottlingHigh
    receiver: "null"
  - match:
      alertname: ^(host_cpu_usage|node_filesystem_free|host_down|TargetDown)$
    receiver: slack_webhook
  - match:
      alertname: SSLCertExpiringSoon
    receiver: slack_webhook
templates:
- /etc/alertmanager/config/*.tmpl
```



现在，当您SSL相关证书还有一个月以内的有效期就会通过Prometheus监控和AlertManager告警通过Slack消息通知到你。

