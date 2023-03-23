---
layout:     post
title:     Kubernetes 多组Pod映射到单个svc服务
subtitle:  Multiple Applications mapped to a single Service
date:       2022-09-07
author:     J
catalog: true
tags:
    - Kubernetes
---

# 背景
Kubernetes的service是一种抽象，它定义了在集群中运行的一组逻辑 Pod，由该组逻辑Pod提供相同的功能。创建时，每个服务都分配有一个唯一的 IP 地址（也称为 clusterIP）。此地址与服务的生命周期相关联，并且在服务存在期间不会更改。Pod可以配置为与Service 通信，此时Service的将自动负载平衡到属于Service成员的其中一个Pod。
>由于service和pod之间通过labels selector 标签过滤器来做关联。

由于label标签的定义可以在`metadata` 和`template`等多个位置，此时需要我们定义在Pod的`template`模板中

您可以使用以下yaml创建`billing-app`和`checkout-app`以及`billing-svc`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: billing-app
spec:
  selector:
    matchLabels:
      run: billing-app
  replicas: 2
  template:
    metadata:
      labels:
        run: billing-app
        component: backend
    spec:
      containers:
      - name: billing-app
        image: nginx
        ports:
        - containerPort: 80
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: checkout-app
spec:
  selector:
    matchLabels:
      run: checkout-app
  replicas: 2
  template:
    metadata:
      labels:
        run: checkout-app
        component: backend
    spec:
      containers:
      - name: checkout-app
        image: nginx
        ports:
        - containerPort: 80
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: billing-service
spec:
  ports:
  - port: 80
    protocol: TCP
  selector:
    component: backend

```

可以看出svc`billing-svc` 通过`selector`来筛选Pod，来实现多组Pod对应一个svc服务
```yaml
...
selector:
  component: backend
...
```
# 验证
```shell
root@node1:~/muti-deplpoyment-a-service# kubectl  get svc
NAME              TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
billing-service   ClusterIP   10.233.2.143   <none>        80/TCP    9s
kubernetes        ClusterIP   10.233.0.1     <none>        443/TCP   195d
root@node1:~/muti-deplpoyment-a-service#
oot@node1:~/muti-deplpoyment-a-service# kubectl  get pod -owide
NAME                            READY   STATUS    RESTARTS   AGE     IP              NODE    NOMINATED NODE   READINESS GATES
billing-app-54c4cb6dc8-fr6tx    1/1     Running   0          2m46s   10.233.66.212   node3   <none>           <none>
billing-app-54c4cb6dc8-wr8lw    1/1     Running   0          2m46s   10.233.68.212   node6   <none>           <none>
checkout-app-66966dfc84-74hxv   1/1     Running   0          2m46s   10.233.67.60    node4   <none>           <none>
checkout-app-66966dfc84-w5689   1/1     Running   0          2m46s   10.233.66.51    node3   <none>           <none>
root@node1:~/muti-deplpoyment-a-service# kubectl  get ep
NAME              ENDPOINTS                                                      AGE
billing-service   10.233.66.212:80,10.233.66.51:80,10.233.67.60:80 + 1 more...   19s
kubernetes        192.168.2.101:6443,192.168.2.102:6443                          195d
```
以上svc`billing-svc`对于的`endpoints`成员包含了4个Pod的ip
