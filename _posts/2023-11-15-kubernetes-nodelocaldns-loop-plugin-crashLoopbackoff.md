---
layout:     post
title:     Kubernetes NodeLocalDNS loop 环形循环导致Pod异常重启
subtitle:  Kubernetes NodeLocalDNS loop loop detected for zone
date:       2023-11-15
author:     J
catalog: true
tags:
    - Kubernetes
    - coreDNS
    - NodeLocalDNS
---

## 背景
近期考虑到自己实验环境在用的Kubernets集群版本还是在`v1.24.4`,已经落后于最新版本`v1.28` 超过3个大版本，所以就准备重新起一个`v1.28`版本的Kubernets的实验环境。
实验环境顺便就把宿主机的OS也升级到了`Debian 12 bookworm`,启用`Linux 6.1.0`的内核，CNI 还是沿用之前一直用的 `cilium`，只是启用了BPF模式，取消`kube-proxy`的iptable的转发数据包的方式。

## 部署
部署工具还是用`Kubespary`
部署过程这里就不赘述了，直接看集群的`Node List`
```
root@node1:~# kubectl get node -o wide
NAME    STATUS   ROLES           AGE     VERSION   INTERNAL-IP     EXTERNAL-IP   OS-IMAGE                         KERNEL-VERSION   CONTAINER-RUNTIME
node1   Ready    control-plane   7d22h   v1.28.3   192.168.2.220   <none>        Debian GNU/Linux 12 (bookworm)   6.1.0-13-amd64   containerd://1.7.7
node2   Ready    <none>          7d22h   v1.28.3   192.168.2.243   <none>        Debian GNU/Linux 12 (bookworm)   6.1.0-13-amd64   containerd://1.7.7
node3   Ready    <none>          7d22h   v1.28.3   192.168.2.222   <none>        Debian GNU/Linux 12 (bookworm)   6.1.0-13-amd64   containerd://1.7.7

root@node1:~# kubectl get pod -A
NAMESPACE            NAME                                          READY   STATUS      RESTARTS      AGE
default              test3                                         1/1     Running     0             124m
ingress-nginx        ingress-nginx-admission-create-j88tn          0/1     Completed   0             6d
ingress-nginx        ingress-nginx-admission-patch-994wx           0/1     Completed   0             6d
ingress-nginx        ingress-nginx-controller-855dc99c4-r9xkh      1/1     Running     2 (28h ago)   6d
kube-system          cilium-bkpln                                  1/1     Running     1 (28h ago)   7d22h
kube-system          cilium-operator-68655fd4f-mcxm6               1/1     Running     2 (28h ago)   7d22h
kube-system          cilium-operator-68655fd4f-n6bsm               1/1     Running     2 (28h ago)   7d22h
kube-system          cilium-qc2p2                                  1/1     Running     1 (28h ago)   7d22h
kube-system          cilium-sp6bl                                  1/1     Running     1 (28h ago)   7d22h
kube-system          coredns-77f7cc69db-58s8b                      1/1     Running     0             58m
kube-system          coredns-77f7cc69db-75wfk                      1/1     Running     0             58m
kube-system          dns-autoscaler-8576bb9f5b-2wc9s               1/1     Running     1 (28h ago)   7d22h
kube-system          kube-apiserver-node1                          1/1     Running     2 (28h ago)   7d22h
kube-system          kube-controller-manager-node1                 1/1     Running     4 (28h ago)   7d22h
kube-system          kube-scheduler-node1                          1/1     Running     2 (28h ago)   7d22h
kube-system          kubernetes-dashboard-759b858758-gtrql         1/1     Running     1 (28h ago)   7d22h
kube-system          kubernetes-metrics-scraper-55bc8b9b54-qmgbj   1/1     Running     1 (28h ago)   7d22h
kube-system          metrics-server-9bbd4847f-sgn8m                1/1     Running     1 (28h ago)   7d22h
kube-system          nginx-proxy-node2                             1/1     Running     1 (28h ago)   7d22h
kube-system          nginx-proxy-node3                             1/1     Running     1 (28h ago)   7d22h
kube-system          node-local-dns-58wlq                          1/1     Running     0             58m
kube-system          node-local-dns-5d5wg                          1/1     Running     0             58m
kube-system          node-local-dns-j4t2l                          1/1     Running     0             58m
local-path-storage   local-path-provisioner-844bd8758f-ntzwx       1/1     Running     1 (28h ago)   7d3h
metallb-system       controller-786f9df989-8crfq                   1/1     Running     2 (28h ago)   6d1h
metallb-system       speaker-9nvbj                                 1/1     Running     3 (28h ago)   6d1h
metallb-system       speaker-c2bdr                                 1/1     Running     2 (28h ago)   6d1h
metallb-system       speaker-swmdd                                 1/1     Running     2 (28h ago)   6d1h

root@node1:~# kubectl exec -it  cilium-bkpln -n kube-system -- cilium status
Defaulted container "cilium-agent" out of: cilium-agent, mount-cgroup (init), apply-sysctl-overwrites (init), clean-cilium-state (init), install-cni-binaries (init)
KVStore:                 Ok   etcd: 1/1 connected, lease-ID=52798bcc5ebf57af, lock lease-ID=52798bcc5ebf57b9, has-quorum=true: https://192.168.2.220:2379 - 3.5.9 (Leader)
Kubernetes:              Ok   1.28 (v1.28.3) [linux/amd64]
Kubernetes APIs:         ["cilium/v2::CiliumClusterwideNetworkPolicy", "cilium/v2::CiliumNetworkPolicy", "core/v1::Namespace", "core/v1::Node", "core/v1::Pods", "core/v1::Service", "discovery/v1::EndpointSlice", "networking.k8s.io/v1::NetworkPolicy"]
KubeProxyReplacement:    Strict   [ens33 192.168.2.220]
Host firewall:           Disabled
CNI Chaining:            none
CNI Config file:         CNI configuration file management disabled
Cilium:                  Ok   1.13.4 (v1.13.4-4061cdfc)
NodeMonitor:             Disabled
Cilium health daemon:    Ok   
IPAM:                    IPv4: 4/254 allocated from 10.233.64.0/24, 
IPv6 BIG TCP:            Disabled
BandwidthManager:        Disabled
Host Routing:            BPF
Masquerading:            BPF   [ens33]   10.233.64.0/24 [IPv4: Enabled, IPv6: Disabled]
Controller Status:       34/34 healthy
Proxy Status:            OK, ip 10.233.64.182, 0 redirects active on ports 10000-20000
Global Identity Range:   min 256, max 65535
Hubble:                  Disabled
Encryption:              Disabled
Cluster health:          3/3 reachable   (2023-11-15T10:27:04Z)

```
可以从Pod的列表里看出，在启用Cilium的`KubeProxyReplacement=Strict` 以及`Host Routing=BPF`和`Masquerading=BPF`的特性之后，kubernetes的核心组件就少了`Kube-proxy`,取而代之是通过eBPF的实现数据包的转发。

## 场景描述
在意外的一次Node节点的重启之后，`NodeLocalDNS`的3个pod相继`CrashLookBackoff`不停地重启，不止如此，还有更严重的是整个集群在新创建Pod时候，都会报`ImagePullBlackOff`的报错，通过logs的日志发现在拉取镜像的时候，DNS无法正常的解析，Pod的dns是指向NodeLocalDNS的`169.254.25.10`这个IP地址,再通过`forward . 10.233.0.3`指向`CoreDNS`的service ip，而CorDNS在解析外部DNS记录的时候，是通过Node自身的`/etc/resolv.conf`这个配置文件中的指向的`nameserver` 这个值。

## NodeLocalDNS 与 CoreDNS的工作机制
`NodeLocalDNS` Pod 运行一个 DNS 服务器，用于侦听来自节点上 Pod 的 DNS 请求。当 Pod 发出 DNS 请求时，请求首先发送到绑定到 IP 地址的本地 `NodeLocalDNS` 实例169.254.20.10。如果本地缓存中存在请求的记录，`NodeLocalDNS` 将缓存结果返回给 pod。如果缓存中不存在该记录，`NodeLocalDNS` 会将请求发送到 CoreDNS Pod，检索相应的 IP 地址，缓存结果，然后将其返回到 Pod。


## 查找问题的原因
在了解了NodeLocalDNS的工作机制之后，我们可以知道，在拉取镜像的时候，是需要通过解析外部DNS记录的，所以需要CoreDNS去请求`UPSTREAM SERVER`上游DNS服务器的，也是Node的`/etc/resolv.conf`中指向的`nameserver`，顺着DNS解析的路径一步步排查之后，发现Node上的`/etc/resolv.conf`配置被`KubeSpary`修改之后变成以下的内容：
```
root@node3:~# cat /etc/resolv.conf
domain cluster.local
search cluster.local default.svc.cluster.local. svc.cluster.local.
nameserver 169.254.25.10
```
这样就能解释为什么在请求外部DNS记录的时候，会报dns无法解析的报错，在CoreDNS的上游DNS服务器也指向`169.254.25.10`的时候，整个DNS的解析就变成了一个无限循环，才会导致`ImagePullBlackOff`拉取镜像的失败。

接下来就关键字搜索查到

- NodeLocalDNS Loop detected for zone "." [https://github.com/kubernetes-sigs/kubespray/issues/9948](https://github.com/kubernetes-sigs/kubespray/issues/9948)

- 关于Kubespary的dns-stack的文档[https://github.com/kubernetes-sigs/kubespray/blob/master/docs/dns-stack.md](https://github.com/kubernetes-sigs/kubespray/blob/master/docs/dns-stack.md)

```
resolvconf_mode: host_resolvconf (default)
This activates the classic Kubespray behavior that modifies the hosts /etc/resolv.conf file and dhclient configuration to point to the cluster dns server (either coredns or coredns_dual, depending on dns_mode).

As cluster DNS is not available on early deployment stage, this mode is split into 2 stages. In the first stage (dns_early: true), /etc/resolv.conf is configured to use the DNS servers found in upstream_dns_servers and nameservers. Later, /etc/resolv.conf is reconfigured to use the cluster DNS server first, leaving the other nameservers as backups.

Also note, existing records will be purged from the /etc/resolv.conf, including resolvconf's base/head/cloud-init config files and those that come from dhclient.
```

在这段描述中可以看到kubespary会去修改`/etc/resolv.conf`,让其指向`CoreDNS`或者`NodeLocalDNS`的地址，这个过程分2个阶段 早期阶段是正常使用`UPSTREAM SERVER`用于解析外部DNS记录的,待集群部署稳定之后，指向集群的NodeLocalDNS,并把`/etc/resolv.conf`做了相应的备份，这也解释了为什么在部署完成，集群稳定之后没有问题，但是一旦Node节点发生重启之后，就无法正常解析外部DNS记录。

查阅了该配置项的目的是在于Node主机能够解析kubernetes的内部的域名服务，而不需要知道服务的 IP 地址，就能访问集群内部域名服务。

后续Kubespary的通过这个*默认不删除默认搜索域*PR修复[https://github.com/kubernetes-sigs/kubespray/pull/10554](https://github.com/kubernetes-sigs/kubespray/pull/10554) 

## 措施

在找到问题的原因之后，就可以着手去解决问题：

- 对于正在运行的集群来说，如果你不需要Node去解析kubernetes内部域名服务的，那我们首要的就是去恢复Node的`/etc/resolv.conf`中的默认配置，然后针对CoreDNS和NodelocalDNS的configmap中对于loop的插件的禁用
- 对于正在运行的集群，你需要Node能解析kubernetes内部域名服务的，那就需要修改coreDNS的configmap中`. forward`配置指向上游DNS服务器,NodeLocalDNS的configmap的`.:53`中`forward . 8.8.8.8 8.8.4.4`，而不是`/etc/resolv.conf`
- 对于需要新建的集群来说，需要我们指定`upstream_dns_server` 为一个非空列表即可。






