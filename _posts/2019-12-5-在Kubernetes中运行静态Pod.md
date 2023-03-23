---
title: 在Kubernetes中运行静态Pod
date: 2019-12-5 11:23
tag: kubernetes
---

**静态**Pod是直接有Node节点上的Kubelet进程进行管理，不通过Master节点Apiserver进行生命周期管理。静态Pod不关联任何Replication Controller控制器，它由Kubelet进程自己来监控，Pod崩溃时Kubelet负责重启Pod，但是静态Pod是没有健康检查，而且始终绑定节点Kubelet进程，所以始终运行在同一个节点，没有办法做迁移。

在此基础上，Kubelet会自动为每个静态Pod在Kubernetes的Apiserver上创建一个与之镜像的Pod（Mirror Pod），此镜像Pod只能在Apiserver上查询状态，但是不能被Apiserver做其他操作（例如：删除）

> 静态Pod被用于Master组件的运行，并配置为Kubelet守护进程的启动/重新加载时运行，Kubeadm在创建集群时，Apiserver都没启动，会使用静态Pod创建master主要组件

静态Pod的创建方式有2种：

- 配置文件
- 通过HTTP

### 配置文件

1. **修改Kubelet.service的启动参数**

常用manifest的YAML文件目录

```
/etc/kubernetes/manifests
```

在/etc/systemd/system/kubelet.service文件下

```shell
# cat kubelet.service 
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
WorkingDirectory=/var/lib/kubelet
ExecStartPre=/bin/mount -o remount,rw '/sys/fs/cgroup'
ExecStartPre=/bin/mkdir -p /sys/fs/cgroup/cpuset/system.slice/kubelet.service
ExecStartPre=/bin/mkdir -p /sys/fs/cgroup/hugetlb/system.slice/kubelet.service
ExecStartPre=/bin/mkdir -p /sys/fs/cgroup/memory/system.slice/kubelet.service
ExecStartPre=/bin/mkdir -p /sys/fs/cgroup/pids/system.slice/kubelet.service
ExecStart=/opt/kube/bin/kubelet \
  --config=/var/lib/kubelet/config.yaml \
  --cni-bin-dir=/opt/kube/bin \
  --cni-conf-dir=/etc/cni/net.d \
  --hostname-override=192.168.2.10 \
  --kubeconfig=/etc/kubernetes/kubelet.kubeconfig \
  --network-plugin=cni \
  --pod-infra-container-image=mirrorgooglecontainers/pause-amd64:3.1 \
  --root-dir=/var/lib/kubelet \
  --cpu-cfs-quota=false \
  --pod-manifest-path=/etc/kubernetes/manifests \ #static pod manifest
  --v=2
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

然后重新启动Kubelet 进程

```shell
systemctl daemon-reload
systemctl restart kubelet.service
```

2. **修改kubelet的config文件`/var/lib/kubelet/config.yaml`**

```yaml
vi /var/lib/kubelet/config.yaml
...
resolvConf: /run/systemd/resolve/resolv.conf
runtimeRequestTimeout: 2m0s
serializeImagePulls: true
# add staticPodPath
staticPodPath: /etc/kubernetes/manifests  
streamingConnectionIdleTimeout: 4h0m0s
...
```

![](https://i.loli.net/2019/12/05/PamYF9KNCz7eDBc.png)

重启kubelet进程

```
systemctl restart kubelet
```

### 通过HTTP创建静态Pods

Kubelet周期地从`--manifest-url=`参数指定的地址下载文件，并且把它翻译成JSON/YAML格式的pod定义。此后的操作方式与`--pod-manifest-path=`相同，kubelet会不时地重新下载该文件，当文件变化时对应地终止或启动静态pod

### 静态pods的动态增加和删除

运行中的kubelet周期扫描配置的目录（我们这个例子中就是`/etc/kubernetes/manifests`）下文件的变化，当这个目录中有文件出现或消失时创建或删除pods

```
[joe@my-node1 ~] $ mv /etc/kubernetes/manifests/static-web.yaml /tmp
[joe@my-node1 ~] $ sleep 20
[joe@my-node1 ~] $ docker ps
// no nginx container is running
[joe@my-node1 ~] $ mv /tmp/static-web.yaml  /etc/kubernetes/manifests/
[joe@my-node1 ~] $ sleep 20
[joe@my-node1 ~] $ docker ps
CONTAINER ID        IMAGE         COMMAND                CREATED           ...
e7a62e3427f1        nginx:latest  "nginx -g 'daemon of   27 seconds ago
```

