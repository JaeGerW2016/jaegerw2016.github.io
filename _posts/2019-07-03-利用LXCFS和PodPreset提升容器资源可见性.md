---
title: 利用LXCFS和PodPreset提升容器资源可见性
date: 2019-07-03 11:54:00
tags: kubernetes
---

### LXCFS简介

社区中常见的做法是利用 [lxcfs](https://yq.aliyun.com/go/articleRenderRedirect?url=https://github.com/lxc/lxcfs)来提供容器中的资源可见性。lxcfs 是一个开源的FUSE（用户态文件系统）实现来支持LXC容器，它也可以支持Docker容器。

LXCFS通过用户态文件系统，在容器中提供下列 `procfs` 的文件。

```
/proc/cpuinfo
/proc/diskstats
/proc/meminfo
/proc/stat
/proc/swaps
/proc/uptime
```

LXCFS的示意图如下

![image](https://i.loli.net/2019/07/03/5d1c254e208b578889.png)

比如，把宿主机的 `/var/lib/lxcfs/proc/memoinfo` 文件挂载到Docker容器的`/proc/meminfo`位置后。容器中进程读取相应文件内容时，LXCFS的FUSE实现会从容器对应的Cgroup中读取正确的内存限制。从而使得应用获得正确的资源约束设定。



#### lxcfs 的 Kubernetes实践

一些同学问过如何在Kubernetes集群环境中使用lxcfs，我们将给大家一个示例方法供参考。

首先我们要在集群节点上安装并启动lxcfs，我们将用Kubernetes的方式，用利用容器和DaemonSet方式来运行 lxcfs FUSE文件系统。

本文所有示例代码可以通过以下地址从Github上获得

```
git clone https://github.com/denverdino/lxcfs-initializer
cd lxcfs-initializer
```

其manifest文件如下

```
apiVersion: apps/v1beta2
kind: DaemonSet
metadata:
  name: lxcfs
  labels:
    app: lxcfs
spec:
  selector:
    matchLabels:
      app: lxcfs
  template:
    metadata:
      labels:
        app: lxcfs
    spec:
      hostPID: true
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      containers:
      - name: lxcfs
        image: registry.cn-hangzhou.aliyuncs.com/denverdino/lxcfs:3.0.4
        imagePullPolicy: Always
        securityContext:
          privileged: true
        volumeMounts:
        - name: cgroup
          mountPath: /sys/fs/cgroup
        - name: lxcfs
          mountPath: /var/lib/lxcfs
          mountPropagation: Bidirectional
        - name: usr-local
          mountPath: /usr/local
      volumes:
      - name: cgroup
        hostPath:
          path: /sys/fs/cgroup
      - name: usr-local
        hostPath:
          path: /usr/local
      - name: lxcfs
        hostPath:
          path: /var/lib/lxcfs
          type: DirectoryOrCreate
```

**注：** 由于 lxcfs FUSE需要共享系统的PID名空间以及需要特权模式，所有我们配置了相应的容器启动参数。

可以通过如下命令在所有集群节点上自动安装、部署完成 lxcfs，是不是很简单？:-)

```
kubectl apply -f lxcfs-daemonset.yaml
```

那么如何在Kubernetes中使用 lxcfs 呢？和上文一样，我们可以在Pod的定义中添加对 /proc 下面文件的 volume（文件卷）和对 volumeMounts（文件卷挂载）定义。然而这就让K8S的应用部署文件变得比较复杂，有没有办法让系统自动完成相应文件的挂载呢？

我们需要使用preset设置一下lxcfs挂载目录即可。

为了在集群中使用Pod Preset: 

1. 在apiserver参数配置preset相关参数,需要修改/etc/kubernetes/manifests/kube-apiserver.yaml文件
2. 二进制形式运行，需要新增`- --enable-admission-plugins=NodeRestriction,PodPreset` `- --runtime-config=settings.k8s.io/v1alpha1=true`

> 可能在某些情况下，您希望 Pod 不会被任何 Pod Preset 突变所改变。在这些情况下，您可以在 Pod 的 Pod Spec 中添加注释： podpreset.admission.kubernetes.io/exclude：”true”。

```yaml
apiVersion: settings.k8s.io/v1alpha1
kind: PodPreset
metadata:
  name: allow-lxcfs
  namespace: default
spec:
  volumeMounts:
    - mountPath: /proc/cpuinfo
      name: lxcfs-cpuinfo
      readOnly: false
    - mountPath: /proc/diskstats
      name: lxcfs-diskstats
      readOnly: false
    - mountPath: /proc/meminfo
      name: lxcfs-meminfo
      readOnly: false
    - mountPath: /proc/stat
      name: lxcfs-stat
      readOnly: false
    - mountPath: /proc/swaps
      name: lxcfs-swaps
      readOnly: false
    - mountPath: /proc/uptime
      name: lxcfs-uptime
      readOnly: false
  volumes:
    - name: lxcfs-cpuinfo
      hostPath:
        path: /var/lib/lxcfs/proc/cpuinfo
    - name: lxcfs-diskstats
      hostPath:
        path: /var/lib/lxcfs/proc/diskstats
    - name: lxcfs-meminfo
      hostPath:
        path: /var/lib/lxcfs/proc/meminfo
    - name: lxcfs-stat
      hostPath:
        path: /var/lib/lxcfs/proc/stat
    - name: lxcfs-swaps
      hostPath:
        path: /var/lib/lxcfs/proc/swaps
    - name: lxcfs-uptime
      hostPath:
        path: /var/lib/lxcfs/proc/uptime
```

这样在default命名空间使用`allow-lxcfs`的PodPreset，会使得该命名空间下pod都挂载lxcfs目录

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: web
  name: web
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: web
          image: httpd:2.4.32
          imagePullPolicy: Always
          resources:
            requests:
              memory: "256Mi"
              cpu: "500m"
            limits:
              memory: "256Mi"
              cpu: "500m"
```

```shell
#pod web 挂载情况
...
    Mounts:
      /proc/cpuinfo from lxcfs-cpuinfo (rw)
      /proc/diskstats from lxcfs-diskstats (rw)
      /proc/meminfo from lxcfs-meminfo (rw)
      /proc/stat from lxcfs-stat (rw)
      /proc/swaps from lxcfs-swaps (rw)
      /proc/uptime from lxcfs-uptime (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-b46m8 (ro)
...
```

```shell
#进入pod查看内存资源
root@k8s-master-1:/var/lib# kubectl exec -it web-7c7bcc76b4-dg45k sh
# free -m   
             total       used       free     shared    buffers     cached
Mem:           256          7        248          0          0          0
-/+ buffers/cache:          7        248
Swap:            0          0          0

```

#### 存在的问题

> lxcfs daemonset的pod故障恢复，如何自动remount已经存在的container的lxcfs相关路径

### 总结

本文介绍通过 lxcfs 和PodPreset提供容器资源可见性的方法，可以帮助一些遗留系统更好的识别容器运行时的资源限制。

Kubernetes 自身还可以通过initializer扩展机制，可以用于对**资源创建**进行拦截和注入处理，我们可以借助它优雅地完成对lxcfs文件的自动化挂载。

这边基于PodPreset的简单易用，没有考虑Initializer的扩展机制。
