---
title: Kubernetes StatefulSet的滚动升级
date: 2019-10-24 16:53
tag: kubernetes
---

## 扩容/缩容 StatefulSet

扩容/缩容 StatefulSet 指增加或减少它的副本数。这通过更新 `replicas` 字段完成。你可以使用[`kubectl scale`](https://kubernetes.io/docs/user-guide/kubectl/v1.16/#scale) 或者[`kubectl patch`](https://kubernetes.io/docs/user-guide/kubectl/v1.16/#patch)来扩容/缩容一个 StatefulSet。

### 扩容

在一个终端窗口观察 StatefulSet 的 Pod。

```shell
kubectl get pods -w -l app=nginx
```

在另一个终端窗口使用 `kubectl scale` 扩展副本数为5。

```shell
kubectl scale sts web --replicas=5
statefulset "web" scaled
```

在第一个 终端中检查 `kubectl get` 命令的输出，等待增加的3个 Pod 的状态变为 Running 和 Ready。

```shell
kubectl get pods -w -l app=nginx
NAME      READY     STATUS    RESTARTS   AGE
web-0     1/1       Running   0          2h
web-1     1/1       Running   0          2h
NAME      READY     STATUS    RESTARTS   AGE
web-2     0/1       Pending   0          0s
web-2     0/1       Pending   0         0s
web-2     0/1       ContainerCreating   0         0s
web-2     1/1       Running   0         19s
web-3     0/1       Pending   0         0s
web-3     0/1       Pending   0         0s
web-3     0/1       ContainerCreating   0         0s
web-3     1/1       Running   0         18s
web-4     0/1       Pending   0         0s
web-4     0/1       Pending   0         0s
web-4     0/1       ContainerCreating   0         0s
web-4     1/1       Running   0         19s
```

StatefulSet 控制器扩展了副本的数量。如同[创建 StatefulSet](https://kubernetes.io/zh/docs/tutorials/stateful-application/basic-stateful-set/#顺序创建pod)所述，StatefulSet 按序号索引顺序的创建每个 Pod，并且会等待前一个 Pod 变为 Running 和 Ready 才会启动下一个Pod。

### 缩容

在一个终端观察 StatefulSet 的 Pod。

```shell
kubectl get pods -w -l app=nginx
```

在另一个终端使用 `kubectl patch` 将 StatefulSet 缩容回三个副本。

```shell
kubectl patch sts web -p '{"spec":{"replicas":3}}'
"web" patched
```

等待 `web-4` 和 `web-3` 状态变为 Terminating。

```
kubectl get pods -w -l app=nginx
NAME      READY     STATUS              RESTARTS   AGE
web-0     1/1       Running             0          3h
web-1     1/1       Running             0          3h
web-2     1/1       Running             0          55s
web-3     1/1       Running             0          36s
web-4     0/1       ContainerCreating   0          18s
NAME      READY     STATUS    RESTARTS   AGE
web-4     1/1       Running   0          19s
web-4     1/1       Terminating   0         24s
web-4     1/1       Terminating   0         24s
web-3     1/1       Terminating   0         42s
web-3     1/1       Terminating   0         42s
```

### 顺序终止 Pod

控制器会按照与 Pod 序号索引相反的顺序每次删除一个 Pod。在删除下一个 Pod 前会等待上一个被完全关闭。

获取 StatefulSet 的 PersistentVolumeClaims。

```shell
kubectl get pvc -l app=nginx
NAME        STATUS    VOLUME                                     CAPACITY   ACCESSMODES   AGE
www-web-0   Bound     pvc-15c268c7-b507-11e6-932f-42010a800002   1Gi        RWO           13h
www-web-1   Bound     pvc-15c79307-b507-11e6-932f-42010a800002   1Gi        RWO           13h
www-web-2   Bound     pvc-e1125b27-b508-11e6-932f-42010a800002   1Gi        RWO           13h
www-web-3   Bound     pvc-e1176df6-b508-11e6-932f-42010a800002   1Gi        RWO           13h
www-web-4   Bound     pvc-e11bb5f8-b508-11e6-932f-42010a800002   1Gi        RWO           13h
```

五个 PersistentVolumeClaims 和五个 PersistentVolumes 仍然存在。查看 Pod 的 [稳定存储](https://kubernetes.io/zh/docs/tutorials/stateful-application/basic-stateful-set/#stable-storage)，我们发现当删除 StatefulSet 的 Pod 时，挂载到 StatefulSet 的 Pod 的 PersistentVolumes不会被删除。当这种删除行为是由 StatefulSe t缩容引起时也是一样的。

## 更新 StatefulSet

Kubernetes 1.7 版本的 StatefulSet 控制器支持自动更新。更新策略由 StatefulSet API Object 的 `spec.updateStrategy` 字段决定。这个特性能够用来更新一个 StatefulSet 中的 Pod 的 container images, resource requests，以及 limits, labels 和 annotations。

### On Delete 策略

`OnDelete` 更新策略实现了传统（1.7之前）行为，它也是默认的更新策略。当你选择这个更新策略并修改 StatefulSet 的 `.spec.template` 字段时， StatefulSet 控制器将不会自动的更新Pod。

Patch `web` StatefulSet 的容器镜像。

```shell
kubectl patch statefulset web --type='json' -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/image", "value":"k8s.gcr.io/nginx-slim:0.7"}]'
"web" patched
```

删除 `web-0` Pod。

```shell
kubectl delete pod web-0
pod "web-0" deleted
```

<– Watch the `web-0` Pod, and wait for it to transition to Running and Ready. –>

观察 `web-0` Pod， 等待它变成 Running 和 Ready。

```shell
kubectl get pod web-0 -w
NAME      READY     STATUS    RESTARTS   AGE
web-0     1/1       Running   0          54s
web-0     1/1       Terminating   0         1m
web-0     0/1       Terminating   0         1m
web-0     0/1       Terminating   0         1m
web-0     0/1       Terminating   0         1m
web-0     0/1       Pending   0         0s
web-0     0/1       Pending   0         0s
web-0     0/1       ContainerCreating   0         0s
web-0     1/1       Running   0         3s
```

获取 `web` StatefulSet 的 Pod 来查看他们的容器镜像。

```shell
kubectl get pod -l app=nginx -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[0].image}{"\n"}{end}'
web-0   k8s.gcr.io/nginx-slim:0.7
web-1   k8s.gcr.io/nginx-slim:0.8
web-2   k8s.gcr.io/nginx-slim:0.8
```

`web-0` 已经更新了它的镜像，但是 `web-1` 和 `web-2` 仍保留了原始镜像。

```shell
kubectl delete pod web-1 web-2
pod "web-1" deleted
pod "web-2" deleted
```

观察 StatefulSet 的 Pod，等待它们全部变成 Running 和 Ready。

```shell
kubectl get pods -w -l app=nginx
NAME      READY     STATUS    RESTARTS   AGE
web-0     1/1       Running   0          8m
web-1     1/1       Running   0          4h
web-2     1/1       Running   0          23m
NAME      READY     STATUS        RESTARTS   AGE
web-1     1/1       Terminating   0          4h
web-1     1/1       Terminating   0         4h
web-1     0/1       Pending   0         0s
web-1     0/1       Pending   0         0s
web-1     0/1       ContainerCreating   0         0s
web-2     1/1       Terminating   0         23m
web-2     1/1       Terminating   0         23m
web-1     1/1       Running   0         4s
web-2     0/1       Pending   0         0s
web-2     0/1       Pending   0         0s
web-2     0/1       ContainerCreating   0         0s
web-2     1/1       Running   0         36s
```

获取 Pod 来查看他们的容器镜像。

```shell
kubectl get pod -l app=nginx -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[0].image}{"\n"}{end}'
web-0   k8s.gcr.io/nginx-slim:0.7
web-1   k8s.gcr.io/nginx-slim:0.7
web-2   k8s.gcr.io/nginx-slim:0.7
```

现在，StatefulSet 中的 Pod 都已经运行了新的容器镜像。

### Rolling Update 策略

`RollingUpdate` 更新策略会更新一个 StatefulSet 中所有的 Pod，采用与序号索引相反的顺序并遵循 StatefulSet 的保证。

Patch `web` StatefulSet 来执行 `RollingUpdate` 更新策略。

```shell
kubectl patch statefulset web -p '{"spec":{"updateStrategy":{"type":"RollingUpdate"}}}'
statefulset "web" patched
```

在一个终端窗口中 patch `web` StatefulSet 来再次的改变容器镜像。

```shell
kubectl patch statefulset web --type='json' -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/image", "value":"k8s.gcr.io/nginx-slim:0.8"}]'
statefulset "web" patched
```

在另一个终端监控 StatefulSet 中的 Pod。

```shell
kubectl get po -l app=nginx -w
NAME      READY     STATUS    RESTARTS   AGE
web-0     1/1       Running   0          7m
web-1     1/1       Running   0          7m
web-2     1/1       Running   0          8m
web-2     1/1       Terminating   0         8m
web-2     1/1       Terminating   0         8m
web-2     0/1       Terminating   0         8m
web-2     0/1       Terminating   0         8m
web-2     0/1       Terminating   0         8m
web-2     0/1       Terminating   0         8m
web-2     0/1       Pending   0         0s
web-2     0/1       Pending   0         0s
web-2     0/1       ContainerCreating   0         0s
web-2     1/1       Running   0         19s
web-1     1/1       Terminating   0         8m
web-1     0/1       Terminating   0         8m
web-1     0/1       Terminating   0         8m
web-1     0/1       Terminating   0         8m
web-1     0/1       Pending   0         0s
web-1     0/1       Pending   0         0s
web-1     0/1       ContainerCreating   0         0s
web-1     1/1       Running   0         6s
web-0     1/1       Terminating   0         7m
web-0     1/1       Terminating   0         7m
web-0     0/1       Terminating   0         7m
web-0     0/1       Terminating   0         7m
web-0     0/1       Terminating   0         7m
web-0     0/1       Terminating   0         7m
web-0     0/1       Pending   0         0s
web-0     0/1       Pending   0         0s
web-0     0/1       ContainerCreating   0         0s
web-0     1/1       Running   0         10s
```

StatefulSet 里的 Pod 采用和序号相反的顺序更新。在更新下一个 Pod 前，StatefulSet 控制器终止每个 Pod 并等待它们变成 Running 和 Ready。请注意，虽然在顺序后继者变成 Running 和 Ready 之前 StatefulSet 控制器不会更新下一个 Pod，但它仍然会重建任何在更新过程中发生故障的 Pod， 使用的是它们当前的版本。已经接收到更新请求的 Pod 将会被恢复为更新的版本，没有收到请求的 Pod 则会被恢复为之前的版本。像这样，控制器尝试继续使应用保持健康并在出现间歇性故障时保持更新的一致性。

获取 Pod 来查看他们的容器镜像。

```shell
for p in 0 1 2; do kubectl get po web-$p --template '{{range $i, $c := .spec.containers}}{{$c.image}}{{end}}'; echo; done
k8s.gcr.io/nginx-slim:0.8
k8s.gcr.io/nginx-slim:0.8
k8s.gcr.io/nginx-slim:0.8
```

StatefulSet 中的所有 Pod 现在都在运行之前的容器镜像。

#### 分段更新

你可以使用 `RollingUpdate` 更新策略的 `partition` 参数来分段更新一个 StatefulSet。分段的更新将会使 StatefulSet 中的其余所有 Pod 保持当前版本的同时仅允许改变 StatefulSet 的 `.spec.template`。

Patch `web` StatefulSet 来对 `updateStrategy` 字段添加一个分区。

```shell
kubectl patch statefulset web -p '{"spec":{"updateStrategy":{"type":"RollingUpdate","rollingUpdate":{"partition":3}}}}'
statefulset "web" patched
```

再次 Patch StatefulSet 来改变容器镜像。

```shell
kubectl patch statefulset web --type='json' -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/image", "value":"k8s.gcr.io/nginx-slim:0.7"}]'
statefulset "web" patched
```

删除 StatefulSet 中的 Pod。

```shell
kubectl delete po web-2
pod "web-2" deleted
```

等待 Pod 变成 Running 和 Ready。

```shell
kubectl get po -lapp=nginx -w
NAME      READY     STATUS              RESTARTS   AGE
web-0     1/1       Running             0          4m
web-1     1/1       Running             0          4m
web-2     0/1       ContainerCreating   0          11s
web-2     1/1       Running   0         18s
```

获取 Pod 的容器。

```shell
get po web-2 --template '{{range $i, $c := .spec.containers}}{{$c.image}}{{end}}'
k8s.gcr.io/nginx-slim:0.8
```

请注意，虽然更新策略是 `RollingUpdate`，StatefulSet 控制器还是会使用原始的容器恢复 Pod。这是因为 Pod 的序号比 `updateStrategy` 指定的 `partition` 更小。

#### 灰度扩容

你可以通过减少 [上文](https://kubernetes.io/zh/docs/tutorials/stateful-application/basic-stateful-set/#分段更新)指定的 `partition` 来进行灰度扩容，以此来测试你的程序的改动。

Patch StatefulSet 来减少分区。

```shell
kubectl patch statefulset web -p '{"spec":{"updateStrategy":{"type":"RollingUpdate","rollingUpdate":{"partition":2}}}}'
statefulset "web" patched
```

等待 `web-2` 变成 Running 和 Ready。

```shell
kubectl get po -lapp=nginx -w
NAME      READY     STATUS              RESTARTS   AGE
web-0     1/1       Running             0          4m
web-1     1/1       Running             0          4m
web-2     0/1       ContainerCreating   0          11s
web-2     1/1       Running   0         18s
```

获取 Pod 的容器。

```shell
kubectl get po web-2 --template '{{range $i, $c := .spec.containers}}{{$c.image}}{{end}}'
k8s.gcr.io/nginx-slim:0.7
```

当你改变 `partition` 时，StatefulSet 会自动的更新 `web-2` Pod，这是因为 Pod 的序号小于或等于 `partition`。

删除 `web-1` Pod。

```shell
kubectl delete po web-1
pod "web-1" deleted
```

等待 `web-1` 变成 Running 和 Ready。

```shell
kubectl get po -lapp=nginx -w
NAME      READY     STATUS        RESTARTS   AGE
web-0     1/1       Running       0          6m
web-1     0/1       Terminating   0          6m
web-2     1/1       Running       0          2m
web-1     0/1       Terminating   0         6m
web-1     0/1       Terminating   0         6m
web-1     0/1       Terminating   0         6m
web-1     0/1       Pending   0         0s
web-1     0/1       Pending   0         0s
web-1     0/1       ContainerCreating   0         0s
web-1     1/1       Running   0         18s
```

获取 `web-1` Pod 的容器。

```shell
get po web-1 --template '{{range $i, $c := .spec.containers}}{{$c.image}}{{end}}'
k8s.gcr.io/nginx-slim:0.8
```

`web-1` 被按照原来的配置恢复，因为 Pod 的序号小于分区。当指定了分区时，如果更新了 StatefulSet 的 `.spec.template`，则所有序号大于或等于分区的 Pod 都将被更新。如果一个序号小于分区的 Pod 被删除或者终止，它将被按照原来的配置恢复。

#### 分阶段的扩容

你可以使用类似[灰度扩容](https://kubernetes.io/zh/docs/tutorials/stateful-application/basic-stateful-set/#灰度扩容)的方法执行一次分阶段的扩容（例如一次线性的、等比的或者指数形式的扩容）。要执行一次分阶段的扩容，你需要设置 `partition` 为希望控制器暂停更新的序号。

分区当前为`2`。请将分区设置为`0`。

```shell
kubectl patch statefulset web -p '{"spec":{"updateStrategy":{"type":"RollingUpdate","rollingUpdate":{"partition":0}}}}'
statefulset "web" patched
```

等待 StatefulSet 中的所有 Pod 变成 Running 和 Ready。

```shell
kubectl get po -lapp=nginx -w
NAME      READY     STATUS              RESTARTS   AGE
web-0     1/1       Running             0          3m
web-1     0/1       ContainerCreating   0          11s
web-2     1/1       Running             0          2m
web-1     1/1       Running   0         18s
web-0     1/1       Terminating   0         3m
web-0     1/1       Terminating   0         3m
web-0     0/1       Terminating   0         3m
web-0     0/1       Terminating   0         3m
web-0     0/1       Terminating   0         3m
web-0     0/1       Terminating   0         3m
web-0     0/1       Pending   0         0s
web-0     0/1       Pending   0         0s
web-0     0/1       ContainerCreating   0         0s
web-0     1/1       Running   0         3s
```

获取 Pod 的容器。

```shell
for p in 0 1 2; do kubectl get po web-$p --template '{{range $i, $c := .spec.containers}}{{$c.image}}{{end}}'; echo; done
k8s.gcr.io/nginx-slim:0.7
k8s.gcr.io/nginx-slim:0.7
k8s.gcr.io/nginx-slim:0.7
```

将 `partition` 改变为 `0` 以允许StatefulSet控制器继续更新过程