---
title: 使用Kubernetes进行零停机滚动更新
date: 2019-03-07 01:26:15
tags: kubernetes
---
软件世界比以往任何时候都更快。为了保持竞争力，需要尽快推出新的软件版本，而不会中断活跃用户访问，影响用户体验。越来越多企业已将其应用迁移到Kubernetes，而Kubernetes的构建基于生产准备。但是，为了通过Kubernetes实现真正的零停机时间，我们需要采取更多步骤，而不会破坏或丢失任何一个用户的请求

关于如何使用Kubernetes实现零停机时间的系列文章

### [滚动更新](https://kubernetes.io/docs/tutorials/kubernetes-basics/update/update-intro/)

默认情况下，Kubernetes部署应用使用滚动更新策略对pod进行更新。此策略旨在通过在执行更新时在任何时间点保证至少一定数量pod实例正常运行来防止应用程序停机。只有在新部署版本的新pod启动并准备好处理流量后，才会关闭旧pod。
工程师可以进一步指定Kubernetes在更新期间如何处理多个副本的确切方式。根据我们可能要配置的工作负载和可用计算资源，我们希望在任何时候不出现过多或不足的实例的现象。例如，给定三个所需的副本，我们应该立即创建三个新的pod并等待它们全部启动，我们应该终止除了一个之外的所有旧pod，还是逐个进行转换？
以下deployment的部分yaml文件显示了具有默认RollingUpdate升级策略的应用程序的Kubernetes部署定义，以及`maxSurge`在更新期间最多一个过度配置的pod数量和` maxUnavailable`最大不可用的pod。
```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: myapp-without-hook
spec:
  replicas: 3
  strategy:
    rollingUpdate:
       maxSurge: 1
       maxUnavailable: 0
...
```
此部署配置将按以下方式执行版本更新过程：它将一次启动一个新版本的pod，等待pod启动并准备就绪，触发其中一个旧pod的终止，然后继续下一个新的pod，直到所有副本都已完成更新。
>为了告诉Kubernetes我们的pod正在运行并准备好处理流量，我们需要配置[存活探针与就绪探针](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/)。

更新过程
```
root@k8s-master-1:~/k8s_manifests/zero-downtime-tutorial# kubectl get pod
myapp-without-hook-77448c5b9f-648h5            1/1     Running             0          76s
myapp-without-hook-77448c5b9f-b9rxm            1/1     Running             0          76s
myapp-without-hook-77448c5b9f-gbrvc            1/1     Running             0          76s
myapp-without-hook-789fd7c548-m5bpw            0/1     ContainerCreating   0          2s
myapp-without-hook-77448c5b9f-648h5            1/1     Terminating         0          79s
...
myapp-without-hook-77448c5b9f-b9rxm            1/1     Running             0          83s
myapp-without-hook-77448c5b9f-gbrvc            1/1     Terminating         0          83s
myapp-without-hook-789fd7c548-79h9d            1/1     Running             0          5s
myapp-without-hook-789fd7c548-chbwh            0/1     ContainerCreating   0          1s
myapp-without-hook-789fd7c548-m5bpw            1/1     Running             0          9s
...
myapp-without-hook-77448c5b9f-b9rxm            1/1     Running             0          85s
myapp-without-hook-77448c5b9f-gbrvc            0/1     Terminating         0          85s
myapp-without-hook-789fd7c548-79h9d            1/1     Running             0          7s
myapp-without-hook-789fd7c548-chbwh            0/1     ContainerCreating   0          3s
myapp-without-hook-789fd7c548-m5bpw            1/1     Running             0          11s
...
myapp-without-hook-77448c5b9f-b9rxm            0/1     Terminating   0          91s
myapp-without-hook-77448c5b9f-gbrvc            0/1     Terminating   0          91s
myapp-without-hook-789fd7c548-79h9d            1/1     Running       0          13s
myapp-without-hook-789fd7c548-chbwh            1/1     Running       0          9s
myapp-without-hook-789fd7c548-m5bpw            1/1     Running       0          17s
```
### 负载压测可用性差距
如果我们执行从旧版本到新版本的滚动更新，并按照pod处于活动状态并准备就绪的输出，则首先行为似乎是有效的。但是，正如我们所看到的，从旧版本到新版本的转换并不总是非常顺利，也就是说，应用程序可能会丢失一些客户端的请求。

为了测试，正在请求的连接是否丢失，特别是那些针对正在停止使用的实例的请求，我们可以使用连接到我们的应用程序的负载测试工具。我们感兴趣的要点是是否所有HTTP请求都得到了正确处理，包括HTTP保持活动连接。为此，我们使用负载压测工具，例如[Apache Bench](http://httpd.apache.org/docs/current/programs/ab.html)或[Fortio](https://github.com/fortio/fortio)。

我们使用多个线程（即多个连接）以并发方式通过HTTP连接到我们正在运行的应用程序。这里我们不关注延迟或吞吐量，而是对响应状态和潜在的连接故障感兴趣。
```
 ⚡ root@ubuntu  ~  fortio load -a -c 8 -qps 500 -t 60s "http://$NodeIP:$NodePort/"
```
这里是用NodePort的服务发布形式，发布我们的示例程序`$NodeIP` `$NodePort` 
```
root@k8s-master-1:~/k8s_manifests/zero-downtime-tutorial# kubectl get svc
myapp-without-hook            NodePort    10.68.21.130    <none>        80:31467/TCP        3s
```
在Fortio的示例中，每秒500个请求和8个线程并发保持活动连接的调用如下所示
```
 ✘ ⚡ root@ubuntu  ~  fortio load -a -c 8 -qps 500 -t 60s "http://192.168.2.12:31467/"
Fortio 1.3.1 running at 500 queries per second, 1->1 procs, for 1m0s: http://192.168.2.12:31467/
13:26:32 I httprunner.go:82> Starting http test for http://192.168.2.12:31467/ with 8 threads at 500.0 qps
Starting at 500 qps with 8 thread(s) [gomax 1] for 1m0s : 3750 calls each (total 30000)
...
Sockets used: 23 (for perfect keepalive, would be 8)
Code  -1 : 4 (0.0 %)
Code 200 : 29996 (100.0 %)
Response Header Sizes : count 30000 avg 235.96853 +/- 2.725 min 0 max 236 sum 7079056
Response Body/Total Sizes : count 30000 avg 330.25453 +/- 3.931 min 0 max 331 sum 9907636
All done 30000 calls (plus 8 warmup) 2.489 ms avg, 500.0 qps
```
>输出表明并非所有请求都可以成功处理code 200。以上的压测结果显示还是有4个请求code -1处理失败。

我们可以运行多个测试场景，通过不同的方式连接到应用程序，例如通过Kubernetes ingress，或通过服务直接从集群内部连接。我们将看到滚动更新期间的行为可能会有所不同，具体取决于我们的测试设置如何连接。与通过入口连接相比，从群集内部连接到服务的客户端可能不会遇到任何数量的失败连接。

### 更新过程发生了什么
现在的问题是，当Kubernetes在滚动更新期间重新路由流量时，从旧的实例版本到新的pod实例版本会发生什么。我们来看看Kubernetes如何管理工作负载连接。

如果我们的客户端，即零停机测试，Fortio直接通过NodeIP:NodePort形式从集群外部连接到服务，它通常使用通过集群DNS解析的服务VIP，最终到达Pod实例，具体的实现通过在每个Kubernetes节点上运行的kube-proxy实现的，并更新iptables转发规则到pod的IP地址。
#### NodePort形式
![image.png](https://upload-images.jianshu.io/upload_images/3481257-b7e085b7b67627ce.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### Ingress的方式
![image.png](https://upload-images.jianshu.io/upload_images/3481257-e2034665ce7a00e5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

无论我们如何连接到我们的应用程序，Kubernetes都旨在最大限度地减少滚动更新过程中的服务中断。

新的pod处于活动状态并正常处理客户端请求，Kubernetes将使旧的pod停止服务，从而更新pod的状态为Terminating，将其从端点对象中删除，然后发送一个`SIGTERM`。在`SIGTERM`导致容器优雅正常退出，并且不接受任何新客户端连接。将pod从endpoints列表中剔除后，负载均衡器会将流量路由到剩余的（新）流量。这就是我们部署中可用性差距的原因; 在终端信号之前，当负载均衡器注意到更改并且可以更新其配置时，已通过`SIGTERM`信号停用Pod。这种重新配置是`异步`发生的，因此不能保证正确的排序，将导致很少的不幸请求被路由到终止pod 从而出现少量的请求丢失现象。

### 走向零停机时间
如何增强我们的应用程序以实现（真正的）零停机时间迁移？

首先，实现这一目标的先决条件是我们的容器正确处理`SIGTERM`信号，即该进程将在Unix上正常退出。看看Google [构建容器](https://cloud.google.com/solutions/best-practices-for-building-containers)[最佳实践](https://cloud.google.com/solutions/best-practices-for-building-containers)如何实现这一目标。

下一步是包括准备探针，检查我们的应用程序是否已准备好处理流量。理想情况下，探针已经检查了需要预热的功能状态，例如高速缓存或servlet初始化。

准备情况探测是我们平滑滚动更新的起点。为了解决pod终端当前没有阻塞的问题并等到负载均衡器重新配置，我们将包含一个`preStop`生命周期钩子。在容器终止之前调用此挂钩。
#### [容器生命周期钩子](https://kubernetes.io/docs/concepts/containers/container-lifecycle-hooks/)
生命周期钩子是同步的，因此必须在最终终止信号被发送到容器之前完成。在我们的例子中，我们使用这个钩子来简单地等待，然后`SIGTERM`才会终止应用程序进程。同时，Kubernetes将从端点对象中删除pod，因此pod将从我们的负载平衡器中排除。我们的生命周期钩子等待时间可确保在应用程序进程停止之前重新配置负载平衡器。

为了实现此行为，我们在myapp的deployment部署中定义了一个preStop钩子，依赖于您选择的技术如何实现就绪和存活探针以及生命周期钩子行为; 
```
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: myapp
    spec:
      terminationGracePeriodSeconds: 10
      containers:
      - name: nginx
        #image: 314315960/zero-downtime-tutorial:green
        image: 314315960/zero-downtime-tutorial:blue
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 80
        readinessProbe:
          httpGet:
            path: /healthy.html
            port: 80
          periodSeconds: 1
          successThreshold: 1
          failureThreshold: 2
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "rm /usr/share/nginx/html/healthy.html && sleep 10"]
```
`readinessProbe` 探针检查该pod是否准备就绪
`proStop` 钩子必须在删除容器的调用之前完成，这里选择删除/healthy.html探针及程序睡10秒的以提供同步宽限期。只有在完成这一系列操作，pod才会继续正常退出。

```
root@k8s-master-1:~/k8s_manifests/zero-downtime-tutorial# kubectl get deploy,svc,pod
NAME                                                READY   UP-TO-DATE   AVAILABLE   AGE
deployment.extensions/myapp                         3/3     3            3           6m38s

NAME                                  TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
myapp                         NodePort    10.68.57.14     <none>        80:26532/TCP        7m32s

service/myapp                         NodePort    10.68.57.14     <none>        80:26532/TCP        6m31s
NAME                                               READY   STATUS    RESTARTS   AGE
pod/myapp-56646bbf86-8dw9f                         1/1     Running   0          6m38s
pod/myapp-56646bbf86-mbzgj                         1/1     Running   0          6m38s
pod/myapp-56646bbf86-mpwk2                         1/1     Running   0          6m38s

#更换image: 314315960/zero-downtime-tutorial:green
root@k8s-master-1:~/k8s_manifests/zero-downtime-tutorial# kubectl edit deployment myapp
deployment.extensions/myapp edited

```
#### Fortio 压测
```
 ? root@ubuntu ? ~ ? fortio load -a -c 8 -qps 500 -t 60s "http://192.168.2.12:26532/"
Fortio 1.3.1 running at 500 queries per second, 1->1 procs, for 1m0s: http://192.168.2.12:26532/
14:11:27 I httprunner.go:82> Starting http test for http://192.168.2.12:26532/ with 8 threads at 500.0 qps
Starting at 500 qps with 8 thread(s) [gomax 1] for 1m0s : 3750 calls each (total 30000)
...
Sockets used: 17 (for perfect keepalive, would be 8)
Code 200 : 30000 (100.0 %)
Response Header Sizes : count 30000 avg 236 +/- 0 min 236 max 236 sum 7080000
Response Body/Total Sizes : count 30000 avg 330.2126 +/- 0.9771 min 329 max 331 sum 9906378
All done 30000 calls (plus 8 warmup) 2.332 ms avg, 499.9 qps


```
>输出表明所有30000个请求都可以成功处理code 200。

![image.png](https://upload-images.jianshu.io/upload_images/3481257-0a3a29f24be36704.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![image.png](https://upload-images.jianshu.io/upload_images/3481257-8e00866adddb34bd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 总结
Kubernetes在编写具有生产准备的应用程序方面做得非常出色。然而，为了在生产中运行我们的企业系统，我们的工程师必须了解Kubernetes如何在引擎盖下运行以及我们的应用程序在启动和关闭期间的行为过程。
#### 以上用到的yaml和dockerfile，都已经上传到[github](https://github.com/JaeGerW2016/k8s_manifests/tree/master/zero-downtime-tutorial)

参考文档：
https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/
https://kubernetes.io/docs/concepts/containers/container-lifecycle-hooks/
https://kubernetes.io/docs/tutorials/kubernetes-basics/update/update-intro/
https://blog.sebastian-daschner.com/entries/zero-downtime-updates-kubernetes
https://github.com/chrismoos/zero-downtime-tutorial

