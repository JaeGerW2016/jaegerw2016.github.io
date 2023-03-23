---
layout:     post
title:     使用 Minikube 创建 Kubernetes 集群
subtitle:  Creating Kubernetes Cluster With Minikube
date:       2022-11-10
author:     J
catalog: true
tags:
    - Kubernetes
---

## 什么是 Minikube？

Minikube 是一个帮助应用程序轻松设置 Kubernetes 集群。不建议用于生产用途。它仅用于开发目的。

它在单个本地环境中提供集群。我们不需要超过一台服务器。

```
user@localhost:~$ minikube start
😄  Ubuntu 22.04 上的 minikube v1.28.0
✨  自动选择 docker 驱动。其他选项：virtualbox, ssh, none
📌  Using Docker driver with root privileges
👍  Starting control plane node minikube in cluster minikube
🚜  Pulling base image ...
💾  Downloading Kubernetes v1.25.3 preload ...
    > preloaded-images-k8s-v18-v1...:  385.44 MiB / 385.44 MiB  100.00% 2.22 Mi
    > gcr.io/k8s-minikube/kicbase:  386.27 MiB / 386.27 MiB  100.00% 2.12 MiB p
    > gcr.io/k8s-minikube/kicbase:  0 B [______________________] ?% ? p/s 1m37s
🔥  Creating docker container (CPUs=2, Memory=3900MB) ...
🐳  正在 Docker 20.10.20 中准备 Kubernetes v1.25.3…
    ▪ Generating certificates and keys ...
    ▪ Booting up control plane ...
    ▪ Configuring RBAC rules ...
🔎  Verifying Kubernetes components...
    ▪ Using image gcr.io/k8s-minikube/storage-provisioner:v5
🌟  Enabled addons: storage-provisioner, default-storageclass
🏄  Done! kubectl is now configured to use "minikube" cluster and "default" namespace by default

```
如果您有服务器或虚拟机，则可以轻松安装 Minikube 并创建集群。

- 首先安装一个容器运行时（例如：Docker、Podman 等）
- 安装 Kubectl
- 安装 Minikube
- 使用 Minikube 创建集群

## 安装 Docker

请根据您的操作系统按照说明进行操作。
https://docs.docker.com/engine/install/

## 安装 Kubectl

请根据您的操作系统按照说明进行操作。

https://kubernetes.io/docs/tasks/tools/

我的服务器是 Ubuntu，所以我关注以下页面：

```shell

curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

```
检查您安装的Kubectl版本：

```
user@localhost:~$ kubectl version --client
WARNING: This version information is deprecated and will be replaced with the output from kubectl version --short.  Use --output=yaml|json to get the full version.
Client Version: version.Info{Major:"1", Minor:"25", GitVersion:"v1.25.3", GitCommit:"434bfd82814af038ad94d62ebe59b133fcb50506", GitTreeState:"clean", BuildDate:"2022-10-12T10:57:26Z", GoVersion:"go1.19.2", Compiler:"gc", Platform:"linux/amd64"}
```

## 安装 Minikube


```
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
```

## 创建 Kubernetes 集群


```
user@localhost:~$ minikube start
```

## 检查minikube 状态
```
user@localhost:~$ minikube status
minikube
type: Control Plane
host: Running
kubelet: Running
apiserver: Running
kubeconfig: Configured

user@localhost:~$ kubectl get  node
NAME       STATUS   ROLES           AGE   VERSION
minikube   Ready    control-plane   32s   v1.25.3

user@localhost:~$ kubectl get  pod --all-namespaces
NAMESPACE     NAME                               READY   STATUS    RESTARTS   AGE
kube-system   coredns-565d847f94-k64q5           1/1     Running   0          31s
kube-system   etcd-minikube                      1/1     Running   0          47s
kube-system   kube-apiserver-minikube            1/1     Running   0          49s
kube-system   kube-controller-manager-minikube   1/1     Running   0          47s
kube-system   kube-proxy-5ws96                   1/1     Running   0          30s
kube-system   kube-scheduler-minikube            1/1     Running   0          45s
kube-system   storage-provisioner                1/1     Running   0          40s
```
