---
layout: mypost
title: Victoria-Metrics-Single Can be Used As a Drop in Replacement For Prometheus in Kubernetes
categories: [Victoria-Metrics, Kubernetes, Prometheus]
---


## 背景介绍

VictoriaMetrics是一种高效、经济且可扩展的监控解决方案，旨在解决Prometheus在数据存储和管理方面的局限性。作为一个时间序列数据库，VictoriaMetrics不仅可以作为Prometheus的长期远程存储，还能独立运行，提供强大的监控功能。其设计理念是优化数据收集和查询性能，使其在处理大规模数据时表现出色，成为云原生环境中不可或缺的工具。

VictoriaMetrics的设计充分考虑了与Prometheus的兼容性，支持Prometheus的API和PromQL查询语言。这种无缝集成的能力使得用户可以轻松地将VictoriaMetrics替换为Prometheus，而无需对现有的监控基础设施进行大规模的重构。这种灵活性为用户提供了更高的选择自由度，能够根据实际需求进行调整。


## 性能比较

作为之前Prometheus的忠实用户, Prometheus在资源受限的环境中,消耗的资源比较高, 所以在看到VictoriaMetrics在数据压缩方面的表现尤为突出，其采用的高效压缩算法使得存储需求显著降低。根据相关研究，VictoriaMetrics的压缩比率可达到Prometheus的十倍，这对于需要长期保存数据的场景尤为重要。这种高效的存储方式不仅节省了硬件成本，还提高了数据管理的灵活性，使得用户能够在有限的存储资源下，处理更多的监控数据。

VictoriaMetrics在内存使用效率方面表现优异，能够在相同的硬件条件下处理更多的时间序列数据。其设计理念是最大限度地减少资源消耗，从而提高数据处理能力。这种高效的内存管理使得用户在进行大规模监控时，能够以更低的成本获得更高的性能，尤其是在资源有限的环境中，VictoriaMetrics的优势更加明显。


## 推荐建议

在选择VictoriaMetrics作为Prometheus的替代品时，首先需要全面评估现有的监控需求和基础设施。这包括分析当前的监控指标、数据量、存储需求以及团队的技术能力。VictoriaMetrics作为一个新兴的监控解决方案，具备与Prometheus高度兼容的特性，能够支持Prometheus的配置文件和查询语言（PromQL），这使得迁移过程相对顺利。通过对比两者的性能和功能，团队可以更好地决定是否进行迁移。

确保团队熟悉VictoriaMetrics的配置和操作是成功迁移的关键。VictoriaMetrics的单节点模式安装简单，通常只需一条命令即可完成部署，这对于技术能力较弱的团队尤为重要。此外，VictoriaMetrics在内存占用方面表现优异，官方数据显示其内存使用量可比Prometheus减少约七倍，这为团队在资源管理上提供了更大的灵活性。通过培训和文档支持，团队可以有效减少在迁移过程中可能遇到的问题。


## 配置步骤


由于篇幅限制,我这边就以 VictoriaMetrics singel 单节点模式替代 Prometheus 来实现监控`wmi_exporter`的Metrics指标, 并在Grafana 展示Dashboard


`vmetrics-single-values.yaml`

```
server:

  scrape:
    enabled: true

    extraScrapeConfigs:
      - job_name: 'windows-node'
        static_configs:
        - targets:
          - 10.100.3.20:9128
          - 10.100.3.35:9128
          - 10.100.3.36:9128
          - 10.100.3.28:9128

```

```
# helm template -n vm-single vmsingle -f victoria-metrics-single-value.yaml vm/victoria-metrics-single
---
# Source: victoria-metrics-single/templates/serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  labels: 
    app.kubernetes.io/managed-by: Helm
    helm.sh/chart: victoria-metrics-single-0.13.3
  name: vmsingle-victoria-metrics-single-server
  namespace: vm-single
---
# Source: victoria-metrics-single/templates/scrape-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: vmsingle-victoria-metrics-single-server-scrapeconfig
  namespace: vm-single
  labels: 
    app: server
    app.kubernetes.io/instance: vmsingle
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: victoria-metrics-single
    app.kubernetes.io/version: v1.108.1
    helm.sh/chart: victoria-metrics-single-0.13.3
data:
  scrape.yml: |
    global:
      scrape_interval: 15s
    scrape_configs:
    - job_name: victoriametrics
      static_configs:
      - targets:
        - localhost:8428
    - bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      job_name: kubernetes-apiservers
      kubernetes_sd_configs:
      - role: endpoints
      relabel_configs:
      - action: keep
        regex: default;kubernetes;https
        source_labels:
        - __meta_kubernetes_namespace
        - __meta_kubernetes_service_name
        - __meta_kubernetes_endpoint_port_name
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        insecure_skip_verify: true
    - bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      job_name: kubernetes-nodes
      kubernetes_sd_configs:
      - role: node
      relabel_configs:
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
      - replacement: kubernetes.default.svc:443
        target_label: __address__
      - regex: (.+)
        replacement: /api/v1/nodes/$1/proxy/metrics
        source_labels:
        - __meta_kubernetes_node_name
        target_label: __metrics_path__
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        insecure_skip_verify: true
    - bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      honor_timestamps: false
      job_name: kubernetes-nodes-cadvisor
      kubernetes_sd_configs:
      - role: node
      relabel_configs:
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
      - replacement: kubernetes.default.svc:443
        target_label: __address__
      - regex: (.+)
        replacement: /api/v1/nodes/$1/proxy/metrics/cadvisor
        source_labels:
        - __meta_kubernetes_node_name
        target_label: __metrics_path__
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        insecure_skip_verify: true
    - job_name: kubernetes-service-endpoints
      kubernetes_sd_configs:
      - role: endpoints
      relabel_configs:
      - action: drop
        regex: true
        source_labels:
        - __meta_kubernetes_pod_container_init
      - action: keep_if_equal
        source_labels:
        - __meta_kubernetes_service_annotation_prometheus_io_port
        - __meta_kubernetes_pod_container_port_number
      - action: keep
        regex: true
        source_labels:
        - __meta_kubernetes_service_annotation_prometheus_io_scrape
      - action: replace
        regex: (https?)
        source_labels:
        - __meta_kubernetes_service_annotation_prometheus_io_scheme
        target_label: __scheme__
      - action: replace
        regex: (.+)
        source_labels:
        - __meta_kubernetes_service_annotation_prometheus_io_path
        target_label: __metrics_path__
      - action: replace
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
        source_labels:
        - __address__
        - __meta_kubernetes_service_annotation_prometheus_io_port
        target_label: __address__
      - action: labelmap
        regex: __meta_kubernetes_service_label_(.+)
      - action: replace
        source_labels:
        - __meta_kubernetes_namespace
        target_label: namespace
      - action: replace
        source_labels:
        - __meta_kubernetes_service_name
        target_label: service
      - action: replace
        source_labels:
        - __meta_kubernetes_pod_node_name
        target_label: node
    - job_name: kubernetes-service-endpoints-slow
      kubernetes_sd_configs:
      - role: endpoints
      relabel_configs:
      - action: drop
        regex: true
        source_labels:
        - __meta_kubernetes_pod_container_init
      - action: keep_if_equal
        source_labels:
        - __meta_kubernetes_service_annotation_prometheus_io_port
        - __meta_kubernetes_pod_container_port_number
      - action: keep
        regex: true
        source_labels:
        - __meta_kubernetes_service_annotation_prometheus_io_scrape_slow
      - action: replace
        regex: (https?)
        source_labels:
        - __meta_kubernetes_service_annotation_prometheus_io_scheme
        target_label: __scheme__
      - action: replace
        regex: (.+)
        source_labels:
        - __meta_kubernetes_service_annotation_prometheus_io_path
        target_label: __metrics_path__
      - action: replace
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
        source_labels:
        - __address__
        - __meta_kubernetes_service_annotation_prometheus_io_port
        target_label: __address__
      - action: labelmap
        regex: __meta_kubernetes_service_label_(.+)
      - action: replace
        source_labels:
        - __meta_kubernetes_namespace
        target_label: namespace
      - action: replace
        source_labels:
        - __meta_kubernetes_service_name
        target_label: service
      - action: replace
        source_labels:
        - __meta_kubernetes_pod_node_name
        target_label: node
      scrape_interval: 5m
      scrape_timeout: 30s
    - job_name: kubernetes-services
      kubernetes_sd_configs:
      - role: service
      metrics_path: /probe
      params:
        module:
        - http_2xx
      relabel_configs:
      - action: keep
        regex: true
        source_labels:
        - __meta_kubernetes_service_annotation_prometheus_io_probe
      - source_labels:
        - __address__
        target_label: __param_target
      - replacement: blackbox
        target_label: __address__
      - source_labels:
        - __param_target
        target_label: instance
      - action: labelmap
        regex: __meta_kubernetes_service_label_(.+)
      - source_labels:
        - __meta_kubernetes_namespace
        target_label: namespace
      - source_labels:
        - __meta_kubernetes_service_name
        target_label: service
    - job_name: kubernetes-pods
      kubernetes_sd_configs:
      - role: pod
      relabel_configs:
      - action: drop
        regex: true
        source_labels:
        - __meta_kubernetes_pod_container_init
      - action: keep_if_equal
        source_labels:
        - __meta_kubernetes_pod_annotation_prometheus_io_port
        - __meta_kubernetes_pod_container_port_number
      - action: keep
        regex: true
        source_labels:
        - __meta_kubernetes_pod_annotation_prometheus_io_scrape
      - action: replace
        regex: (.+)
        source_labels:
        - __meta_kubernetes_pod_annotation_prometheus_io_path
        target_label: __metrics_path__
      - action: replace
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
        source_labels:
        - __address__
        - __meta_kubernetes_pod_annotation_prometheus_io_port
        target_label: __address__
      - action: labelmap
        regex: __meta_kubernetes_pod_label_(.+)
      - action: replace
        source_labels:
        - __meta_kubernetes_namespace
        target_label: namespace
      - action: replace
        source_labels:
        - __meta_kubernetes_pod_name
        target_label: pod
    - job_name: windows-node
      static_configs:
      - targets:
        - 10.100.3.20:9128
        - 10.100.3.35:9128
        - 10.100.3.36:9128
        - 10.100.3.28:9128
---
# Source: victoria-metrics-single/templates/clusterrole.yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: vmsingle-victoria-metrics-single-server
  labels: 
    app.kubernetes.io/managed-by: Helm
    helm.sh/chart: victoria-metrics-single-0.13.3
rules:
  - apiGroups:
    - discovery.k8s.io
    resources:
    - endpointslices
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources:
      - nodes
      - nodes/proxy
      - nodes/metrics
      - services
      - endpoints
      - pods
    verbs: ["get", "list", "watch"]
  - apiGroups:
      - networking.k8s.io
    resources:
      - ingresses
    verbs: ["get", "list", "watch"]
  - nonResourceURLs: ["/metrics"]
    verbs: ["get"]
---
# Source: victoria-metrics-single/templates/clusterrolebinding.yaml
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: vmsingle-victoria-metrics-single-server
  labels: 
    app.kubernetes.io/managed-by: Helm
    helm.sh/chart: victoria-metrics-single-0.13.3
subjects:
  - kind: ServiceAccount
    name: vmsingle-victoria-metrics-single-server
    namespace: vm-single
roleRef:
  kind: ClusterRole
  name: vmsingle-victoria-metrics-single-server
  apiGroup: rbac.authorization.k8s.io
---
# Source: victoria-metrics-single/templates/server-service.yaml
apiVersion: v1
kind: Service
metadata:
  namespace: vm-single
  labels: 
    app: server
    app.kubernetes.io/instance: vmsingle
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: victoria-metrics-single
    app.kubernetes.io/version: v1.108.1
    helm.sh/chart: victoria-metrics-single-0.13.3
  name: vmsingle-victoria-metrics-single-server
spec:
  type: ClusterIP
  clusterIP: None
  ports:
    - name: http
      port: 8428
      protocol: TCP
      targetPort: http
  selector: 
    app: server
    app.kubernetes.io/instance: vmsingle
    app.kubernetes.io/name: victoria-metrics-single
---
# Source: victoria-metrics-single/templates/server-statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  namespace: vm-single
  labels: 
    app: server
    app.kubernetes.io/instance: vmsingle
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: victoria-metrics-single
    app.kubernetes.io/version: v1.108.1
    helm.sh/chart: victoria-metrics-single-0.13.3
  name: vmsingle-victoria-metrics-single-server
spec:
  serviceName: vmsingle-victoria-metrics-single-server
  selector:
    matchLabels: 
      app: server
      app.kubernetes.io/instance: vmsingle
      app.kubernetes.io/name: victoria-metrics-single
  replicas: 1
  podManagementPolicy: OrderedReady
  template:
    metadata:
      labels: 
        app: server
        app.kubernetes.io/instance: vmsingle
        app.kubernetes.io/managed-by: Helm
        app.kubernetes.io/name: victoria-metrics-single
        app.kubernetes.io/version: v1.108.1
        helm.sh/chart: victoria-metrics-single-0.13.3
    spec:
      containers:
        - name: vmsingle
          securityContext: 
            {}
          image: victoriametrics/victoria-metrics:v1.108.1
          imagePullPolicy: IfNotPresent
          args: 
            - --envflag.enable
            - --envflag.prefix=VM_
            - --httpListenAddr=:8428
            - --loggerFormat=json
            - --promscrape.config=/scrapeconfig/scrape.yml
            - --retentionPeriod=1
            - --storageDataPath=/storage
          ports:
            - name: http
              containerPort: 8428
          readinessProbe: 
            failureThreshold: 3
            httpGet:
              path: /health
              port: http
              scheme: HTTP
            initialDelaySeconds: 5
            periodSeconds: 15
            timeoutSeconds: 5
          livenessProbe: 
            failureThreshold: 10
            initialDelaySeconds: 30
            periodSeconds: 30
            tcpSocket:
              port: http
            timeoutSeconds: 5
          volumeMounts:
            - name: server-volume
              mountPath: /storage
            - name: scrapeconfig
              mountPath: /scrapeconfig
            
      securityContext: 
        {}
      serviceAccountName: vmsingle-victoria-metrics-single-server
      automountServiceAccountToken: true
      terminationGracePeriodSeconds: 60
      volumes:
        - name: scrapeconfig
          configMap:
            name: vmsingle-victoria-metrics-single-server-scrapeconfig
        
  volumeClaimTemplates:
    - apiVersion: v1
      kind: PersistentVolumeClaim
      metadata:
        name: server-volume
      spec:
        accessModes: 
          - ReadWriteOnce
        resources:
          requests:
            storage: 16Gi
```

> Tips: 如果需要新增额外的ScrapeConfigs时,直接修改configmap无法被VictoriaMetrics识别生效,这时需要先定义`extraScrapeConfigs.yaml`, 然后`helm upgrade --install -n vm-single vmsingle vm/victoria-metrics-single --set server.scrape.enable=true --set-file server.scrape.extraScrapeConfigs=extraScrapeConfigs.yaml` 来追加额外的监控项

`grafana-value.yaml`

```
  datasources:
    datasources.yaml:
      apiVersion: 1
      datasources:
        - name: victoriametrics
          type: prometheus
          orgId: 1
          url: http://vmsingle-victoria-metrics-single-server.vm-single.svc.cluster.local:8428
          access: proxy
          isDefault: true
          updateIntervalSeconds: 10
          editable: true

  dashboardProviders:
   dashboardproviders.yaml:
     apiVersion: 1
     providers:
     - name: 'default'
       orgId: 1
       folder: ''
       type: file
       disableDeletion: true
       editable: true
       options:
         path: /var/lib/grafana/dashboards/default

  dashboards:
    default:
      victoriametrics:
        gnetId: 10229
        revision: 22
        datasource: victoriametrics
      kubernetes:
        gnetId: 14205
        revision: 1
        datasource: victoriametrics
```

```
helm install vm-grafana grafana/grafana -n vm-single -f grafana-value.yaml --dry-run > grafana-manifest.yaml
kubectl apply -f grafana-manifest.yaml

```

```
# kubectl get pod -n vm-single
NAME                                        READY   STATUS    RESTARTS   AGE
vm-grafana-56c64d96c8-5m7qd                 1/1     Running   0          9d
vmsingle-victoria-metrics-single-server-0   1/1     Running   0          5d20h

```

`grafana-ingress.yaml`

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: grafana-ingress
  annotations:
    kubernetes.io/ingress.class: traefik
spec:
  rules:
    - host: grafana.yourowndomain.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: vm-grafana
                port:
                  number: 80
```

`vmetrics-vmui-ingress.yaml`


```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: victoria-metrics-vmui-ingress
  annotations:
    kubernetes.io/ingress.class: traefik
spec:
  rules:
    - host: victoria-metrics-vmui.yourowndomain.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: vmsingle-victoria-metrics-single-server
                port:
                  number: 8428

``` 

```
# kubectl get ingress -n vm-single
NAME                            CLASS    HOSTS                             ADDRESS       PORTS   AGE
grafana-ingress                 <none>   grafana.yourowndomain.com                 10.101.3.31   80      9d
victoria-metrics-vmui-ingress   <none>   victoria-metrics-vmui.yourowndomain.com   10.101.3.31   80      6d
```

### 查看Vmui 中 Target和Service-discovery

VMetrics 自身支持简易的vmui 可以在配置阶段辅助排查问题, 只要你之前对Prometheus的界面熟悉,这个界面也是非常容易上手的


![Picture-20250106122805](https://youjb.com/images/2025/01/06/QQ20250106122805d4bcec1ce7458db6.png)


### 导入并查看Grafana的window exporter的Dashboard


由于vmetrics和prometheus的高度兼容性,可以直接导入以下dashboard的id或者下载的json文件


[Windows Exporter Dashboard 2024](https://grafana.com/grafana/dashboards/20763-windows-exporter-dashboard-2024/)


### Grafana Dashboard About Windows Node Metrics

![Picture-20250106122805](https://youjb.com/images/2025/01/06/QQ20250106122805d4bcec1ce7458db6.png)![Picture 20250106123521](https://youjb.com/images/2025/01/06/Picture-202501061235217d91ad6962344993.png)


## 总结 

以上就是简单的对Prometheus的替换, 在这个过程中可以学习到 Vmetrics 优秀的架构和设计理念, 相信未来Vmetrics 会越来越简单高效 补充Prometheus在监控方面的缺点和不足
