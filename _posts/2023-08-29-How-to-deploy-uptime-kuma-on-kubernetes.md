---
layout: mypost
title: 如何在 Kubernetes 集群上部署 Uptime-Kuma 
categories: [Kubernetes,Uptime-Kuma]
---

Uptime-Kuma 一个开源免费的监控工具 Uptime Kuma, 简单实用, 主要用来监控 Web 和网络, 和 Prometheus 不一样的是, 它是轻量的, 基于Node.js 和 Vue 3 开发, 你可以在 github 找到它

[https://github.com/louislam/uptime-kuma](https://github.com/louislam/uptime-kuma)

主要特性如下:

•可监控 HTTP(s) / TCP / Ping / DNS / Redis / Container等
•支持 Webhook,邮件多种通知方式
•多语言支持•轻量, 基于 Node.js 和 Vue 3 开发
•花哨的、响应式的 Dashboard•开源免费
•支持 Docker 部署

`Uptime Kuma `也提供了在线的演示站点，你可以尝试体验一下。
我创建了一组清单，它将在 `Kubernetes` 集群上部署 `Uptime-Kuma` 以及所有需要的组件。

首先，我们为 `Uptime-Kuma` 创建一个名为`kuma`的`Namespaces` 这里可以根据您的需要更改你想要的。

`Namspace.yaml`
```yaml
kind: Namespace
apiVersion: v1
metadata:
  name: kuma
```

` Uptime-Kuma `创建一个`Service`服务，它将侦听端口`3001`。标签选择器`app: uptime-kuma`

`service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: uptime-kuma-service
  namespace: kuma
spec:
  selector:
    app: uptime-kuma
  ports:
  - name: uptime-kuma
    port: 3001

```
现在是我们的`` Statefulset ``的核心部分了，它将描述实际的` Deployment `和持久卷 `pv`。

`statefulset.yaml`

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: uptime-kuma
  namespace: kuma
spec:
  replicas: 1
  serviceName: uptime-kuma-service
  selector:
    matchLabels:
      app: uptime-kuma
  template:
    metadata:
      labels:
        app: uptime-kuma
    spec:
      containers:
        - name: uptime-kuma
          image: louislam/uptime-kuma:latest
          env:
            - name: UPTIME_KUMA_PORT
              value: "3001"
            - name: PORT
              value: "3001"
          ports:
            - name: uptime-kuma
              containerPort: 3001
              protocol: TCP
          volumeMounts:
            - name: kuma-data
              mountPath: /app/data

  volumeClaimTemplates:
    - metadata:
        name: kuma-data
      spec:
        accessModes: ["ReadWriteOnce"]
        volumeMode: Filesystem
        resources:
          requests:
            storage: 2Gi

```

以上yaml文件中定义了一个`kuma-data`卷pvc定义，系统自行会去找满足条件的`Storageclass`去生成`PV`，状态变更为`Bound`状态

```shell
root@i-ddfdgdx1 ~]# kubectl  get pvc -n kuma
NAME                      STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
kuma-data-uptime-kuma-0   Bound    pvc-b40eaf13-46b5-4def-85c5-9d80f4606e98   5Gi        RWO            local-path     5d19h
```

可以看到这个的`Storageclass`为当前集群里定义的`local-path`，利用Node本地存储来生成PV

部署完以上之后，我们需要给`Uptime-kuma`部署访问入口，通常情况下我们都会通过Ingress来提供。

`Ingress.yaml`

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: kuma-ui
  namespace: kuma
spec:
  rules:
  - host: uptime-kuma.xx.xx
    http:
      paths:
      - backend:
          serviceName: uptime-kuma-service
          servicePort: 3001
        path: /
        pathType: Prefix
status:
  loadBalancer:
    ingress:
    - ip: xx.xx.xx.xx
```

完成以上操作之后能够连接到 Uptime-Kuma。首次登录时，您必须创建一个管理员用户。完成此操作后，您将被重定向到仪表板。与 Uptime-Kuma 一起玩得开心！

![uptime-kuma.png](https://thedatabaseme.de/wp-content/uploads/2022/02/uptime_kuma_dashboard-768x380.png)