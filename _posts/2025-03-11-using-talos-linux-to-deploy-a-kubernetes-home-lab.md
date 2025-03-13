---
layout: mypost
title: 基于Talos Linux创建Kubernetes的Home Lab
categories: [Talos, kubernetes]
---


## 介绍
Talos Linux是一种专为Kubernetes设计的现代操作系统，具有安全、不可变和极简的特性。这使得它成为管理Kubernetes集群的理想选择，特别是在家庭实验室环境中。

### Talos Linux特性

- 安全性：Talos Linux通过不可变的基础设施设计减少了攻击面，并使用相互TLS认证来确保所有API访问的安全性
- 不可变性：该系统以只读模式运行，并移除所有与主机级别相关的交互，如Shell和SSH，进一步提高了安全性
- 极简设计：Talos仅包含运行Kubernetes所需的少量二进制和共享库


### 准备工作

为了在home lab使用Talos Linux创建Kubernetes集群，您需要准备以下硬件和软件：

#### 硬件需求

- Intel NUC等设备
- 树莓派或其他SBC设备
- 支持PXE引导的机器或虚拟机

>确保您的机器具有至少 2 GB 的内存和 2 核心的 CPU。使用多个节点时，需要根据节点数量调整硬件资源。

#### 软件要求

- Docker环境，用于快速设置Talos测试集群
- talosctl工具，Talos的命令行工具
- kubectl工具，用于Kubernetes的命令行操作

>确保您的环境中安装了 Docker 和 Talos CLI 工具，Talos CLI 可以帮助您管理集群和节点。

### Talosctl CLI 安装


```
#macOS homebrew

brew install siderolabs/tap/talosctl

#This script will work on macOS, Linux, and WSL on Windows

curl -sL https://talos.dev/install | sh
```

### Kubectl CLI 安装

**Download the latest release with the command:**

```
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
```



## Talos Linux 安装

这里以VMware workstation的3台虚拟机为示例:


```
Talos version: v.1.9.4
kubernetes version: v1.32.2

controlplane IP : 192.168.2.243
worker IP: 192.168.2.227
worker IP: 192.168.2.230

```

### 下载Talos Linux镜像

ISO镜像地址:[Talos release pages](https://github.com/siderolabs/talos/releases/latest/)

- `metal-<arch>.iso supports` booting on BIOS and UEFI systems (for x86, UEFI only for arm64
- `metal-<arch>-secureboot.iso` supports booting on only UEFI systems in SecureBoot mode (via Image Factory)

在Vmware workstation中的新建虚拟机的步骤就不详细列举了,自行


## 在 Talos 上配置 Kubernetes 集群

```
#!/bin/bash
# Talos Linux deploy kuberetes cluster script

set -e
# env config
CLUSTER_NAME="mycluster"
KUBERNETES_VERSION="v1.13.2"
CONTROLPLANE_NODES="192.168.2.243"
WORKER_NODES="192.168.2.227 192.168.2.230"

# general kubernetes controlplane config
talosctl gen config ${CLUSTER_NAME} https://${CONTROLPLANE_NODES}:6443 --kubernetes-version ${KUBERNETES_VERSION}

# bootstrap kubernetes cluster
talosctl bootstrap -n ${CONTROLPLANE_NODES} -e ${CONTROLPLANE_NODES} --talosconfig ./talosconfig

echo "Waiting for control plane nodes to be ready..."
talosctl --nodes ${CONTROLPLANE_NODES} health --wait-timeout 10m

echo "Applying configuration to worker nodes..."
for node in $WORKER_NODES; do
  echo "Configuring worker node: $node"
  talosctl apply-config --insecure --nodes $node --file worker.yaml
done

# talosctl dashboard status
talos-8a9-rda (v1.9.4): uptime 1h36m21s, 4x2.11GHz, 3.8 GiB RAM, PROCS 34, CPU 10.2%, RAM 20.8%                                                                                                                                                                                                                            
                                                                                                                                                                                                                                                                                                                           
 UUID       6fe14d56-7d61-5ff6-3377-e65c165c2e2e                                                                       TYPE               controlplane                                               HOST         talos-8a9-rda                                                                                            
 CLUSTER    mycluster (3 machines)                                                                                     KUBERNETES         v1.32.2                                                    IP           192.168.2.243/24                                                                                         
 SIDEROLINK n/a                                                                                                        KUBELET            √ Healthy                                                  GW           192.168.2.10                                                                                             
 STAGE      √ Running                                                                                                  APISERVER          √ Healthy                                                  CONNECTIVITY √ OK                                                                                                     
 READY      √ True                                                                                                     CONTROLLER-MANAGER √ Healthy                                                  DNS          192.168.2.1                                                                                              
 SECUREBOOT × False                                                                                                    SCHEDULER          √ Healthy                                                  NTP          time.cloudflare.com   

# general kubeconfig via talosctl
echo "Retrieving kubeconfig..."
talosctl -e ${CONTROLPLANE_NODES} -n ${CONTROLPLANE_NODES} --talosconfig ./talosconfig kubeconfig ./kubeconfig

# export KUBECONFG env
export KUBECONFIG=./kubeconfig

```

## 管理部署应用

在通过talosctl 生成kubeconfig和声明环境变量`KUBECONFIG`的路径之后, 就能通过`kubectl`的CLI命令行工具来管理kubernetes集群了.

```
export KUBECONFIG=./kubeconfig

root@debian:/opt/talos# kubectl cluster-info
Kubernetes control plane is running at https://192.168.2.243:6443
CoreDNS is running at https://192.168.2.243:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
root@debian:/opt/talos# kubectl config current-context
admin@mycluster

root@debian:/opt/talos# kubectl get node -owide
NAME            STATUS   ROLES           AGE   VERSION   INTERNAL-IP     EXTERNAL-IP   OS-IMAGE         KERNEL-VERSION   CONTAINER-RUNTIME
talos-8a9-rda   Ready    control-plane   43h   v1.32.2   192.168.2.243   <none>        Talos (v1.9.4)   6.12.13-talos    containerd://2.0.2
talos-cd5-qf4   Ready    <none>          43h   v1.32.2   192.168.2.227   <none>        Talos (v1.9.4)   6.12.13-talos    containerd://2.0.2
talos-wis-rba   Ready    <none>          43h   v1.32.2   192.168.2.230   <none>        Talos (v1.9.4)   6.12.13-talos    containerd://2.0.2

root@debian:/opt/talos# kubectl get pod -A
NAMESPACE                       NAME                                             READY   STATUS    RESTARTS       AGE
kube-system                     coredns-578d4f8ffc-l72qx                         1/1     Running   1 (107m ago)   43h
kube-system                     coredns-578d4f8ffc-vn4kj                         1/1     Running   1 (107m ago)   43h
kube-system                     kube-apiserver-talos-8a9-rda                     1/1     Running   0              106m
kube-system                     kube-controller-manager-talos-8a9-rda            1/1     Running   2 (106m ago)   106m
kube-system                     kube-flannel-gpv5b                               1/1     Running   1 (117m ago)   43h
kube-system                     kube-flannel-hd58h                               1/1     Running   1 (107m ago)   43h
kube-system                     kube-flannel-z5lwm                               1/1     Running   1 (107m ago)   43h
kube-system                     kube-proxy-62g4z                                 1/1     Running   1 (107m ago)   43h
kube-system                     kube-proxy-b6jgr                                 1/1     Running   1 (107m ago)   43h
kube-system                     kube-proxy-wxf76                                 1/1     Running   1 (117m ago)   43h
kube-system                     kube-scheduler-talos-8a9-rda                     1/1     Running   2 (106m ago)   106m
kube-system                     metrics-server-6f7dd4c4c4-655nm                  0/1     Running   1 (107m ago)   43h
kubelet-serving-cert-approver   kubelet-serving-cert-approver-66ddcd6c99-57mrh   1/1     Running   1 (117m ago)   43h

```

## 总结

通过 Talos Linux 创建 Kubernetes Home Lab 是一项技术性强且富有挑战的任务。得益于 Talos 设计的精简和针对性，它极大地减少了 Kubernetes 的运维复杂度，适合开发者和爱好者在家中的学习和实验。无论是进行小规模的个人项目，还是探索云原生世界，使用 Talos Linux 都能够提供一个高效、安全的环境

## 参考资源

[Talos Linux Doc](https://www.talos.dev/v1.9/introduction/)

[Talos on Intel NUC home lab](https://www.youtube.com/watch?v=h3tQAFImx48)
