---
layout:     post
title:     Kubernetes实践——使用 Argo Rollouts 自动化蓝/绿部署
subtitle:  Automating Blue/Green Deployment with Argo Rollouts
date:       2022-10-23
author:     J
catalog: true
tags:
    - Kubernetes
    - argocd
---

## 什麼是Argo Rollouts
Argo Rollouts 是一个Kubernetes 控制器和一组CRD，为Kubernetes提供高级部署功能，例如蓝绿部署、金丝雀部署、金丝雀分析、更新测试(experimentation)和渐进式交付(progressive delivery)功能。

Argo Rollouts可与入口控制器和服务网格集成，利用它们的流量整形能力在更新期间逐渐将流量转移到新版本。此外，Argo Rollouts 可以查询的各種providers的Metrics指标（可定制的metric查询和kpi分析），并在更新期间進行自动升级或回滚。

## 原理
Argo Rollouts原理和Deployment的差不多，其CRD定義及源碼引用了Deployment一部分數據結構，只是加强rollout的策略和流量控制。当spec.template发送变化时，Argo-Rollout就会根据spec.strategy进行rollout，通常会产生一个新的ReplicaSet，逐步scale down之前的ReplicaSet的pod数量。


## 安裝
```shell
# instal argo rollouts controller and CRD
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml

# install argo rollouts kubectl plugin
curl -LO https://github.com/argoproj/argo-rollouts/releases/latest/download/kubectl-argo-rollouts-linux-amd64
chmod +x ./kubectl-argo-rollouts-linux-amd64
mv ./kubectl-argo-rollouts-linux-amd64 /usr/local/bin/kubectl-argo-rollouts
```

## Argo Rollouts 部署策略
要使用 Argo Rollouts，我们声明一个资源，其apiVersion属性为 `argoproj.io/v1alpha1`，Kind为`Rollout`，如下所示：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
```
Argo Rollouts 的配置有一个`strategy`属性供我们选择我们想要的部署策略，有两个`blueGreen`和`canary`。
在這裏我們先來了解`blueGreen`部署策略（藍/綠部署）。



## 實踐

`blueGreen-rollouts.yaml`

```yaml
# This example demonstrates a Rollout using the blue-green update strategy, which contains a manual
# gate before promoting the new stack.
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: rollout-bluegreen
spec:
  replicas: 2
  revisionHistoryLimit: 2
  selector:
    matchLabels:
      app: rollout-bluegreen
  template:
    metadata:
      labels:
        app: rollout-bluegreen
    spec:
      containers:
      - name: rollouts-demo
        image: argoproj/rollouts-demo:blue
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
  strategy:
    blueGreen:
      # activeService specifies the service to update with the new template hash at time of promotion.
      # This field is mandatory for the blueGreen update strategy.
      activeService: rollout-bluegreen-active
      # previewService specifies the service to update with the new template hash before promotion.
      # This allows the preview stack to be reachable without serving production traffic.
      # This field is optional.
      previewService: rollout-bluegreen-preview
      # autoPromotionEnabled disables automated promotion of the new stack by pausing the rollout
      # immediately before the promotion. If omitted, the default behavior is to promote the new
      # stack as soon as the ReplicaSet are completely ready/available.
      # Rollouts can be resumed using: `kubectl argo rollouts promote ROLLOUT`
      autoPromotionEnabled: false

---
kind: Service
apiVersion: v1
metadata:
  name: rollout-bluegreen-active
spec:
  type: NodePort
  selector:
    app: rollout-bluegreen
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080

---
kind: Service
apiVersion: v1
metadata:
  name: rollout-bluegreen-preview
spec:
  type: NodePort
  selector:
    app: rollout-bluegreen
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
```
Rollout 的數據結構与原生的 Deployment Object 相同，只是`strategy`属性不同。
在上面的文件中，我们声明了 3 个有關於`blueGreen` 的`strategy`属性：
- `autoPromotionEnabled: false`— 表示 rollout 是否应自动将新的 ReplicaSet 提升为`Active`或进入`Pause`状态。如果未指定，则默认值为 true。
- `activeService: brollout-bluegreen-active`— 对部署修改为`Active`服务的服务的引用。
- `previewService: rollout-bluegreen-preview` — 推出修改为预览服务的服务的名称。

以上yaml中创建2個`service`，它們除了`name`属性之外，这两个 Services的selector屬性是一致的。
```yaml
...
selector:
  app: rollout-bluegreen
...
```
创建一个 Rollout 对象和二個 Service 服務
```shell
root@node1:~/argo-rollouts# kubectl apply -f bluegreen-rollout.yaml -n argo-rollouts
rollout.argoproj.io/rollout-bluegreen created
service/rollout-bluegreen-active created
service/rollout-bluegreen-preview created

root@node1:~/argo-rollouts# kubectl get svc -n argo-rollouts
NAME                        TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
argo-rollouts-dashboard     ClusterIP   10.233.46.185   <none>        3100/TCP       2d1h
argo-rollouts-metrics       ClusterIP   10.233.42.13    <none>        8090/TCP       2d5h
rollout-bluegreen-active    NodePort    10.233.15.20    <none>        80:32677/TCP   3m41s
rollout-bluegreen-preview   NodePort    10.233.15.157   <none>        80:31906/TCP   3m40s
root@node1:~/argo-rollouts#

root@node1:~/argo-rollouts# kubectl get rollout -n argo-rollouts
NAME                DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
rollout-bluegreen   2         2         2            2           3m52s

root@node1:~/ephemeral-containers# kubectl exec -it busybox -n default /bin/sh
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.

/ # nslookup rollout-bluegreen-active.argo-rollouts.svc.cluster.local.
Server:    169.254.25.10
Address 1: 169.254.25.10

Name:      rollout-bluegreen-active.argo-rollouts.svc.cluster.local.
Address 1: 10.233.15.20 rollout-bluegreen-active.argo-rollouts.svc.cluster.local

/ # nslookup rollout-bluegreen-preview.argo-rollouts.svc.cluster.local.
Server:    169.254.25.10
Address 1: 169.254.25.10

Name:      rollout-bluegreen-preview.argo-rollouts.svc.cluster.local.
Address 1: 10.233.15.157 rollout-bluegreen-preview.argo-rollouts.svc.cluster.local

root@node1:~/ephemeral-containers# kubectl get rs -n argo-rollouts
NAME                                 DESIRED   CURRENT   READY   AGE
argo-rollouts-76fcfc8d7f             1         1         1       2d6h
argo-rollouts-dashboard-75f845ff57   1         1         1       2d1h
rollout-bluegreen-5ffd47b8d4         2         2         2       27m

root@node1:~/ephemeral-containers# kubectl get pod -n argo-rollouts -owide
NAME                                       READY   STATUS    RESTARTS   AGE    IP              NODE    NOMINATED NODE   READINESS GATES
argo-rollouts-76fcfc8d7f-tnb9w             1/1     Running   0          2d6h   10.233.67.164   node5   <none>           <none>
argo-rollouts-dashboard-75f845ff57-kw64d   1/1     Running   0          2d1h   10.233.68.186   node4   <none>           <none>
rollout-bluegreen-5ffd47b8d4-cp4ws         1/1     Running   0          25m    10.233.67.213   node5   <none>           <none>
rollout-bluegreen-5ffd47b8d4-nb7mh         1/1     Running   0          25m    10.233.68.79    node4   <none>           <none>

root@node1:~/ephemeral-containers# kubectl get ep -n argo-rollouts -owide
NAME                        ENDPOINTS                              AGE
argo-rollouts-dashboard     10.233.68.186:3100                     2d1h
argo-rollouts-metrics       10.233.67.164:8090                     2d6h
rollout-bluegreen-active    10.233.67.213:8080,10.233.68.79:8080   28m
rollout-bluegreen-preview   10.233.67.213:8080,10.233.68.79:8080   28m
```
此时，rollout-bluegreen-active和rollout-bluegreen-preview都指向
同一個`rollout-bluegreen-5ffd47b8d4`的ReplicaSet。

### 打開瀏覽器
訪問`rollout-bluegreen-active`和`rollout-bluegreen-preview`的UI 發現是一樣的服務
[![20221026190815cc6066353683001c.md.jpg](https://youjb.com/images/2022/10/26/20221026190815cc6066353683001c.md.jpg)](https://youjb.com/image/c2S)


现在，我们更改imageRollout Containers Spec image的tag的属性：

```yaml
...
spec:
  containers:
  - name: rollouts-demo
    image: argoproj/rollouts-demo:green
...
```
更新yaml文件apply
```shell

root@node1:~/argo-rollouts# kubectl -n argo-rollouts apply -f bluegreen-rollout.yaml
rollout.argoproj.io/rollout-bluegreen configured
service/rollout-bluegreen-active unchanged
service/rollout-bluegreen-preview unchanged

#此时，Argo Rollouts 将为新yaml文件创建一个新的ReplicaSet: rollout-bluegreen-75695867f
root@node1:~/argo-rollouts# kubectl -n argo-rollouts get rs
NAME                                 DESIRED   CURRENT   READY   AGE
argo-rollouts-76fcfc8d7f             1         1         1       2d7h
argo-rollouts-dashboard-75f845ff57   1         1         1       2d2h
rollout-bluegreen-5ffd47b8d4         2         2         2       85m
rollout-bluegreen-75695867f          2         2         2       74s

#然后将rollout-bluegreen-preview服务修改为指向新的 ReplicaSet,對應的endppoint地址有變化
root@node1:~/argo-rollouts# kubectl -n argo-rollouts get ep
NAME                        ENDPOINTS                              AGE
argo-rollouts-dashboard     10.233.68.186:3100                     2d2h
argo-rollouts-metrics       10.233.67.164:8090                     2d7h
rollout-bluegreen-active    10.233.67.213:8080,10.233.68.79:8080   86m
rollout-bluegreen-preview   10.233.66.4:8080,10.233.69.98:8080     86m


```

访问`rollout-bluegreen-preview`的不同的UI
[![20221026192317d842f80cc930e9cc.md.jpg](https://youjb.com/images/2022/10/26/20221026192317d842f80cc930e9cc.md.jpg)](https://youjb.com/image/c24)

而`rollout-bluegreen-active`服务UI没有变化

使用Argo-Rollout提供的plugin查看其状态：

```shell
root@node1:~/argo-rollouts# kubectl argo rollouts get rollout rollout-bluegreen  -n argo-rollouts
Name:            rollout-bluegreen
Namespace:       argo-rollouts
Status:          ॥ Paused
Message:         BlueGreenPause
Strategy:        BlueGreen
Images:          argoproj/rollouts-demo:blue (stable, active)
                 argoproj/rollouts-demo:green (preview)
Replicas:
  Desired:       2
  Current:       4
  Updated:       2
  Ready:         2
  Available:     2

NAME                                           KIND        STATUS     AGE  INFO
⟳ rollout-bluegreen                            Rollout     ॥ Paused   98m  
├──# revision:2                                                            
│  └──⧉ rollout-bluegreen-75695867f            ReplicaSet  ✔ Healthy  14m  preview
│     ├──□ rollout-bluegreen-75695867f-mhjnv   Pod         ✔ Running  14m  ready:1/1
│     └──□ rollout-bluegreen-75695867f-mng65   Pod         ✔ Running  14m  ready:1/1
└──# revision:1                                                            
   └──⧉ rollout-bluegreen-5ffd47b8d4           ReplicaSet  ✔ Healthy  98m  stable,active
      ├──□ rollout-bluegreen-5ffd47b8d4-cp4ws  Pod         ✔ Running  98m  ready:1/1
      └──□ rollout-bluegreen-5ffd47b8d4-nb7mh  Pod         ✔ Running  98m  ready:1/1
```
接下来，我们通过promote來把rollout-bluegreen服务指向它来 Rollout到ReplicaSet的新版本，我们运行以下命令
```shell
root@node1:~/argo-rollouts# kubectl argo rollouts promote rollout-bluegreen -n argo-rollouts
rollout 'rollout-bluegreen' promoted

```
使用Argo-Rollout提供的plugin查看其状态：

```shell
root@node1:~/argo-rollouts# kubectl argo rollouts get rollout rollout-bluegreen  -n argo-rollouts
Name:            rollout-bluegreen
Namespace:       argo-rollouts
Status:          ✔ Healthy
Strategy:        BlueGreen
Images:          argoproj/rollouts-demo:blue
                 argoproj/rollouts-demo:green (stable, active)
Replicas:
  Desired:       2
  Current:       4
  Updated:       2
  Ready:         2
  Available:     2

NAME                                           KIND        STATUS     AGE   INFO
⟳ rollout-bluegreen                            Rollout     ✔ Healthy  105m  
├──# revision:2                                                             
│  └──⧉ rollout-bluegreen-75695867f            ReplicaSet  ✔ Healthy  20m   stable,active
│     ├──□ rollout-bluegreen-75695867f-mhjnv   Pod         ✔ Running  20m   ready:1/1
│     └──□ rollout-bluegreen-75695867f-mng65   Pod         ✔ Running  20m   ready:1/1
└──# revision:1                                                             
   └──⧉ rollout-bluegreen-5ffd47b8d4           ReplicaSet  ✔ Healthy  104m  delay:8s
      ├──□ rollout-bluegreen-5ffd47b8d4-cp4ws  Pod         ✔ Running  104m  ready:1/1
      └──□ rollout-bluegreen-5ffd47b8d4-nb7mh  Pod         ✔ Running  104m  ready:1/1
```
訪問`rollout-bluegreen-active`和`rollout-bluegreen-preview`的UI 發現是一樣的服務
[![202210261936159bd6976e785c034f.md.jpg](https://youjb.com/images/2022/10/26/202210261936159bd6976e785c034f.md.jpg)](https://youjb.com/image/c2A)

现在，Argo Rollouts 更新rollout-bluegreen服务以指向新的 ReplicaSet，等待（默认 30 秒）后，旧的 ReplicaSet 被缩小

```shell
root@node1:~/argo-rollouts# kubectl argo rollouts get rollout rollout-bluegreen  -n argo-rollouts
Name:            rollout-bluegreen
Namespace:       argo-rollouts
Status:          ✔ Healthy
Strategy:        BlueGreen
Images:          argoproj/rollouts-demo:green (stable, active)
Replicas:
  Desired:       2
  Current:       2
  Updated:       2
  Ready:         2
  Available:     2

NAME                                          KIND        STATUS        AGE   INFO
⟳ rollout-bluegreen                           Rollout     ✔ Healthy     112m  
├──# revision:2                                                               
│  └──⧉ rollout-bluegreen-75695867f           ReplicaSet  ✔ Healthy     27m   stable,active
│     ├──□ rollout-bluegreen-75695867f-mhjnv  Pod         ✔ Running     27m   ready:1/1
│     └──□ rollout-bluegreen-75695867f-mng65  Pod         ✔ Running     27m   ready:1/1
└──# revision:1                                                               
   └──⧉ rollout-bluegreen-5ffd47b8d4          ReplicaSet  • ScaledDown  111m  
```
## 用户界面Dashboard
[![2022102619440207b54226e2b86a1c.md.jpg](https://youjb.com/images/2022/10/26/2022102619440207b54226e2b86a1c.md.jpg)](https://youjb.com/image/c2J)

可以在Dashboard裏做之前命令行進行的Promote


## 结论
用户希望在新版本开始为生产流量提供服务之前对其运行最后一分钟的功能测试。借助 BlueGreen 策略，Argo Rollouts 允许用户指定预览服务和活动服务。Argo Rollouts 将配置预览服务以将流量发送到新版本，而活动服务继续接收生产流量。一旦用户满意，他们可以将预览服务推广为新的活动服务。
本文简单体验了一下Argo Rollouts的蓝/绿部署功能，学会了如何使用 Argo Rollouts 自动化蓝/绿部署，正如你所看到的，它也很简单。
