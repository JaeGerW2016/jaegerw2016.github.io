---
title: Prometheus Operator 学习笔记
date: 2019-01-06 22:29:29
tags:
---
### 文档说明
实验环境：kubernetes Version v1.10.9
网络CNI：fannel
存储CSI: NFS Dynamic Class
DNS: CoreDNS


## 背景
在学习Prometheus Operator 的部署，Prometheus 在代码上就已经对 Kubernetes 有了原生的支持，可以通过服务发现的形式来自动监控集群，因此我们可以使用另外一种更加高级的方式来部署 Prometheus：Operator 框架。
#### Prometheus-Operator的架构图：
![Prometheus-Operator的架构图](https://upload-images.jianshu.io/upload_images/3481257-41bd3ec3de5e831d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

上图是Prometheus-Operator官方提供的架构图，其中Operator是最核心的部分，作为一个控制器，他会去创建`Prometheus`、`ServiceMonitor`、`AlertManage`r以及`PrometheusRule`4个CRD资源对象，然后会一直监控并维持这4个资源对象的状态。

其中创建的prometheus这种资源对象就是作为Prometheus Server存在，而ServiceMonitor就是exporter的各种抽象，exporter前面我们已经学习了，是用来提供专门提供metrics数据接口的工具，Prometheus就是通过ServiceMonitor提供的metrics数据接口去 pull 数据的，当然alertmanager这种资源对象就是对应的AlertManager的抽象，而PrometheusRule是用来被Prometheus实例使用的报警规则文件。

这样我们要在集群中监控什么数据，就变成了直接去操作 Kubernetes 集群的资源对象了，是不是方便很多了。上图中的 `Service` 和 `ServiceMonitor` 都是 Kubernetes 的资源，一个 ServiceMonitor 可以通过 labelSelector 的方式去匹配一类 Service，Prometheus 也可以通过 labelSelector 去匹配多个ServiceMonitor。

## 安装Operator

```
$ git clone https://github.com/coreos/prometheus-operator
$ cd contrib/kube-prometheus/manifests/
$ ls
00namespace-namespace.yaml                                         node-exporter-clusterRole.yaml
0prometheus-operator-0alertmanagerCustomResourceDefinition.yaml    node-exporter-daemonset.yaml
......
```
> `prometheus-serviceMonitorKubelet.yaml` 进行简单的修改，因为默认情况下，这个 ServiceMonitor 是关联的 kubelet 的10250端口去采集的节点数据，而我们前面说过为了安全，这个 metrics 数据已经迁移到10255这个只读端口上,我们只需要将文件中的`https-metrics`更改成`http-metrics`即可

```
root@k8s-master-1:~/k8s_manifests/prometheus-operator# cat prometheus-serviceMonitorKubelet.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  labels:
    k8s-app: kubelet
  name: kubelet
  namespace: monitoring
spec:
  endpoints:
  - bearerTokenFile: /var/run/secrets/kubernetes.io/serviceaccount/token
    honorLabels: true
    interval: 30s
    port: http-metrics
    scheme: http
    tlsConfig:
      insecureSkipVerify: true
  - bearerTokenFile: /var/run/secrets/kubernetes.io/serviceaccount/token
    honorLabels: true
    interval: 30s
    path: /metrics/cadvisor
    port: http-metrics
    scheme: http
    tlsConfig:
      insecureSkipVerify: true
  jobLabel: k8s-app
  namespaceSelector:
    matchNames:
    - kube-system
  selector:
    matchLabels:
      k8s-app: kubelet
```
修改完成后，直接在该文件夹下面执行创建资源命令即可：
```
root@k8s-master-1:~/k8s_manifests/prometheus-operator# kubectl apply -f .
root@k8s-master-1:~/k8s_manifests/prometheus-operator# ls
00namespace-namespace.yaml                                         grafana-dashboardDefinitions.yaml           node-exporter-clusterRole.yaml                       prometheus-clusterRoleBinding.yaml
0prometheus-operator-0alertmanagerCustomResourceDefinition.yaml    grafana-dashboardSources.yaml               node-exporter-clusterRoleBinding.yaml                prometheus-k8s-ingress.yaml
0prometheus-operator-0prometheusCustomResourceDefinition.yaml      grafana-deployment.yaml                     node-exporter-daemonset-017.yaml                     prometheus-kubeSchedulerService.yaml
0prometheus-operator-0prometheusruleCustomResourceDefinition.yaml  grafana-ingress.yaml                        node-exporter-daemonset.yaml                         prometheus-prometheus.yaml
0prometheus-operator-0servicemonitorCustomResourceDefinition.yaml  grafana-service.yaml                        node-exporter-service.yaml                           prometheus-roleBindingConfig.yaml
0prometheus-operator-clusterRole.yaml                              grafana-serviceAccount.yaml                 node-exporter-serviceAccount.yaml                    prometheus-roleBindingSpecificNamespaces.yaml
0prometheus-operator-clusterRoleBinding.yaml                       kube-controller-manager-endpoints.yaml      node-exporter-serviceMonitor.yaml                    prometheus-roleConfig.yaml
0prometheus-operator-deployment.yaml                               kube-controller-manager-service.yaml        prometheus-adapter-apiService.yaml                   prometheus-roleSpecificNamespaces.yaml
0prometheus-operator-service.yaml                                  kube-scheduler-endpoints.yaml               prometheus-adapter-clusterRole.yaml                  prometheus-rules.yaml
0prometheus-operator-serviceAccount.yaml                           kube-scheduler-service.yaml                 prometheus-adapter-clusterRoleBinding.yaml           prometheus-service.yaml
0prometheus-operator-serviceMonitor.yaml                           kube-state-metrics-clusterRole.yaml         prometheus-adapter-clusterRoleBindingDelegator.yaml  prometheus-serviceAccount.yaml
alertmanager-alertmanager.yaml                                     kube-state-metrics-clusterRoleBinding.yaml  prometheus-adapter-clusterRoleServerResources.yaml   prometheus-serviceMonitor.yaml
alertmanager-secret.yaml                                           kube-state-metrics-deployment.yaml          prometheus-adapter-configMap.yaml                    prometheus-serviceMonitorApiserver.yaml
alertmanager-service.yaml                                          kube-state-metrics-role.yaml                prometheus-adapter-deployment.yaml                   prometheus-serviceMonitorCoreDNS.yaml
alertmanager-serviceAccount.yaml                                   kube-state-metrics-roleBinding.yaml         prometheus-adapter-roleBindingAuthReader.yaml        prometheus-serviceMonitorKubeControllerManager.yaml
alertmanager-serviceMonitor.yaml                                   kube-state-metrics-service.yaml             prometheus-adapter-service.yaml                      prometheus-serviceMonitorKubeScheduler.yaml
coredns-metrics-service.yaml                                       kube-state-metrics-serviceAccount.yaml      prometheus-adapter-serviceAccount.yaml               prometheus-serviceMonitorKubelet.yaml
grafana-dashboardDatasources.yaml                                  kube-state-metrics-serviceMonitor.yaml      prometheus-clusterRole.yaml
```
部署完成后，会创建一个名为monitoring的 namespace，所以资源对象对将部署在改命名空间下面，此外 Operator 会自动创建4个 CRD 资源对象：
```
root@k8s-master-1:~/k8s_manifests/prometheus-operator# kubectl get crd |grep coreos
alertmanagers.monitoring.coreos.com     17d
prometheuses.monitoring.coreos.com      17d
prometheusrules.monitoring.coreos.com   17d
servicemonitors.monitoring.coreos.com   17d
```
可以在 monitoring 命名空间下面查看所有的 Pod，其中 alertmanager 和 prometheus 是用 StatefulSet 控制器管理的，其中还有一个比较核心的 prometheus-operator 的 Pod，用来控制其他资源对象和监听对象变化的
```
root@k8s-master-1:~/k8s_manifests/prometheus-operator# kubectl get pods -n monitoring
NAME                                   READY     STATUS    RESTARTS   AGE
alertmanager-main-0                    2/2       Running   4          17d
alertmanager-main-1                    2/2       Running   6          17d
alertmanager-main-2                    2/2       Running   4          17d
grafana-6d6c4d998d-v9djh               1/1       Running   0          8d
kube-state-metrics-5c5c6f7f8f-frwpk    4/4       Running   0          14d
loki-5c5d8d7d7d-gcvcx                  1/1       Running   0          8d
loki-grafana-996d8c8fc-shm29           1/1       Running   0          8d
loki-promtail-cpqq9                    1/1       Running   0          8d
loki-promtail-k786c                    1/1       Running   0          8d
loki-promtail-lmmn2                    1/1       Running   0          8d
loki-promtail-xlb8b                    1/1       Running   0          8d
node-exporter-8gdh4                    2/2       Running   2          14d
node-exporter-cdmbk                    2/2       Running   0          14d
node-exporter-pqzbf                    2/2       Running   0          14d
node-exporter-x4968                    2/2       Running   0          14d
prometheus-adapter-69466cc54b-vgqpg    1/1       Running   2          17d
prometheus-k8s-0                       3/3       Running   8          13d
prometheus-k8s-1                       3/3       Running   3          11d
prometheus-operator-56954c76b5-rjlbq   1/1       Running   0          14d
```
```
root@k8s-master-1:~/k8s_manifests/prometheus-operator# kubectl get svc -n monitoring
NAME                    TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
alertmanager-main       ClusterIP   10.68.209.89    <none>        9093/TCP            17d
alertmanager-operated   ClusterIP   None            <none>        9093/TCP,6783/TCP   17d
grafana                 ClusterIP   10.68.149.168   <none>        3000/TCP            17d
kube-state-metrics      ClusterIP   None            <none>        8443/TCP,9443/TCP   17d
loki                    ClusterIP   10.68.118.118   <none>        3100/TCP            8d
loki-grafana            ClusterIP   10.68.77.53     <none>        80/TCP              8d
node-exporter           ClusterIP   None            <none>        9100/TCP            17d
prometheus-adapter      ClusterIP   10.68.217.16    <none>        443/TCP             17d
prometheus-k8s          ClusterIP   10.68.193.174   <none>        9090/TCP            17d
prometheus-operated     ClusterIP   None            <none>        9090/TCP            17d
prometheus-operator     ClusterIP   None            <none>        8080/TCP            17d
```
部署Ingress 允许外部访问
```
root@k8s-master-1:~/k8s_manifests/prometheus-operator# cat grafana-ingress.yaml 
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: grafana-ui
  namespace: monitoring
spec:
  rules:
  - host: grafana.k8s.io
    http:
      paths:
      - backend:
          serviceName: grafana
          servicePort: service
        path: /
status:
  loadBalancer: {}
```

### 监控二进制组件
由于当前集群的部署方式，Master的核心组件Kube-scheduler和kube-controller-manager是通过二进制文件启动，而不是以Pod的形式，这是一个非常重要的概念
就和 ServiceMonitor 的定义有关系了，我们先来查看下 kube-scheduler 组件对应的 ServiceMonitor 资源的定义：(`prometheus-serviceMonitorKubeScheduler.yaml`)
```
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  labels:
    k8s-app: kube-scheduler
  name: kube-scheduler
  namespace: monitoring
spec:
  endpoints:
  - interval: 30s
    port: http-metrics  
  jobLabel: k8s-app
  namespaceSelector: 
    matchNames:
    - kube-system
  selector: 
    matchLabels:
      k8s-app: kube-scheduler
```
```
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  labels:
    k8s-app: kube-controller-manager
  name: kube-controller-manager
  namespace: monitoring
spec:
  endpoints:
  - interval: 30s
    metricRelabelings:
    - action: drop
      regex: etcd_(debugging|disk|request|server).*
      sourceLabels:
      - __name__
    port: http-metrics
  jobLabel: k8s-app
  namespaceSelector:
    matchNames:
    - kube-system
  selector:
    matchLabels:
      k8s-app: kube-controller-manager
```
上面是一个典型的 ServiceMonitor 资源文件的声明方式，上面我们通过selector.matchLabels在 kube-system 这个命名空间下面匹配具有k8s-app=kube-scheduler这样的 Service，但是我们系统中根本就没有对应的 Service，所以我们需要手动创建一个 Service：
`kube-controller-manager-service.yaml` 
`kube-scheduler-service.yaml`
```
apiVersion: v1
kind: Service
metadata:
  labels:
    k8s-app: kube-scheduler
  name: kube-scheduler
  namespace: kube-system
spec:
  type: ClusterIP
  clusterIP: None
  ports:
  - name: http-metrics
    port: 10251
    protocol: TCP
    targetPort: 10251

```
```
apiVersion: v1
kind: Service
metadata:
  namespace: kube-system
  name: kube-controller-manager
  labels:
    k8s-app: kube-controller-manager
spec:
  type: ClusterIP
  clusterIP: None
  ports:
  - name: http-metrics
    port: 10252
    targetPort: 10252
    protocol: TCP
```
> kube-controller-manager.service 监听的地址改成0.0.0.0
  ExecStart=/opt/kube/bin/kube-controller-manager \
  --address=0.0.0.0 \
  --master=http://0.0.0.0:8080 \
 kube-scheduler.service 监听的地址改成0.0.0.0
  ExecStart=/opt/kube/bin/kube-scheduler \
  --address=0.0.0.0 \
  --master=http://0.0.0.0:8080 \

#### Prometheus 的Targets监控

![Prometheus 的Targets监控](https://upload-images.jianshu.io/upload_images/3481257-fd7d6683f8bc6a54.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### Grafana Dashboard 
![Grafana Dashboard ](https://upload-images.jianshu.io/upload_images/3481257-3d68071b03ab2b90.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
