---
layout:     post
title:     Kubernetes故障排除——如何调试CoreDNS及CoreDNS性能优化
subtitle:  How to Debug CoreDNS Issues And Performance Optimization
date:       2022-11-29
author:     J
catalog: true
tags:
    - Kubernetes
    - CoreDNS
---

## 概述
CoreDNS 作为 Kubernetes 集群的域名解析组件，它可以用来为集群内部提供域名解析服务，在 Kubernetes 1.11 中，CoreDNS 已经达到基于 DNS 服务发现的 General Availability (GA)。


在本文中，我将分享一些我经常用来排查 K8s CoreDNS 问题的技巧。

### 检查Node级别的 DNS 服务

```shell
root@node1:~# kubectl get svc -n kube-system -l k8s-app=kube-dns
NAME       TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)                  AGE
coredns    ClusterIP   10.233.0.3    <none>        53/UDP,53/TCP,9153/TCP   81d
kube-dns   ClusterIP   10.233.0.10   <none>        53/UDP,53/TCP,9153/TCP   76d

# kube-dns 和 coredns 使用不同svc 但是指向后端相同的pod和endpoint

root@node1:~# nc -vz 10.233.0.3 53
coredns.kube-system.svc.cluster.local [10.233.0.3] 53 (domain) open
root@node1:~# nc -vz 10.233.0.10 53
kube-dns.kube-system.svc.cluster.local [10.233.0.10] 53 (domain) open
```
如果您无法连接到DNS集群 ip，请检查您的`iptables`或者`ipvs`规则，这个取决于你`kube-proxy`您应该会看到类似的规则：
```
#kube-proxy use iptables

-A KUBE-SERVICES -d 10.233.0.3/32 -p tcp -m comment --comment "kube-system/kube-dns:metrics cluster IP" -m tcp --dport 9153 -j KUBE-SVC-JD5MR3NA4I4DYORP
-A KUBE-SERVICES -d 10.233.0.3/32 -p udp -m comment --comment "kube-system/kube-dns:dns cluster IP" -m udp --dport 53 -j KUBE-SVC-TCOU7JCQXEZGVUNU
-A KUBE-SERVICES -d 10.233.0.3/32 -p tcp -m comment --comment "kube-system/kube-dns:dns-tcp cluster IP" -m tcp --dport 53 -j KUBE-SVC-ERIFXISQEP7F7OF4

#kube-proxy use ipvs
root@node1:~# ipvsadm -Ln | grep -A2 -E "10.233.0.3|10.233.0.10"
TCP  10.233.0.3:53 rr
  -> 10.233.64.148:53             Masq    1      0          0         
  -> 10.233.65.103:53             Masq    1      0          0         
TCP  10.233.0.3:9153 rr
  -> 10.233.64.148:9153           Masq    1      0          0         
  -> 10.233.65.103:9153           Masq    1      0          0         
TCP  10.233.0.10:53 rr
  -> 10.233.64.148:53             Masq    1      0          0         
  -> 10.233.65.103:53             Masq    1      0          0         
TCP  10.233.0.10:9153 rr
  -> 10.233.64.148:9153           Masq    1      0          0         
  -> 10.233.65.103:9153           Masq    1      0          0         
--
UDP  10.233.0.3:53 rr
  -> 10.233.64.148:53             Masq    1      0          0         
  -> 10.233.65.103:53             Masq    1      0          0         
UDP  10.233.0.10:53 rr
  -> 10.233.64.148:53             Masq    1      0          0         
  -> 10.233.65.103:53             Masq    1      0          0     

```

### 网络排查工具容器

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: busybox
  namespace: default
spec:
  containers:
  - image: busybox:1.28
    name: busybox
    command:
      - sleep
      - "3600"
    imagePullPolicy: IfNotPresent
  restartPolicy: Always

```

```shell
root@node1:~/ephemeral-containers# kubectl apply -f pod.yaml
root@node1:~/ephemeral-containers# kubectl get pod -n default
NAME      READY   STATUS              RESTARTS   AGE
busybox   0/1     ContainerCreating   0          7s
root@node1:~/ephemeral-containers#
root@node1:~/ephemeral-containers# kubectl get pod -n default
NAME      READY   STATUS    RESTARTS   AGE
busybox   1/1     Running   0          62s
root@node1:~/ephemeral-containers#
root@node1:~/ephemeral-containers# kubectl exec -it busybox -n default -- /bin/sh
/ #
/ # nslookup kubernetes.default
Server:    169.254.25.10
Address 1: 169.254.25.10

Name:      kubernetes.default
Address 1: 10.233.0.1 kubernetes.default.svc.cluster.local
/ # nslookup google.com
Server:    169.254.25.10
Address 1: 169.254.25.10

Name:      google.com
Address 1: 142.251.42.238 tsa01s11-in-f14.1e100.net

/ # cat /etc/resolv.conf
search default.svc.cluster.local svc.cluster.local cluster.local
nameserver 169.254.25.10
options ndots:5
```

您还可以尝试在`control-plane`节点上运行这个 `busybox `pod，以检查 DNS 服务是否在主节点上正常工作：
这个是时候需要添加`tolerations`配置

```yaml
...
  tolerations:
  - key: "node-role.kubernetes.io/control-plane"
    effect: "NoSchedule"
  nodeSelector:
    kubernetes.io/hostname: <node_name>

```

### 检查 DNS Pod 运行状况
您可以使用kubectl get pods命令来验证 DNS pod 是否正在运行：
```shell
root@node1:~/ephemeral-containers# kubectl get po -n kube-system -l k8s-app=kube-dns
NAME                       READY   STATUS    RESTARTS     AGE
coredns-74d6c5659f-24q2g   1/1     Running   6 (8d ago)   76d
coredns-74d6c5659f-9vh7w   1/1     Running   3 (8d ago)   37d
```
如果所有CoreDNSPod 都运行良好，您可以使用kubectl `logs`命令查看来自 DNS 容器的日志：

```shell
root@node1:~/ephemeral-containers# kubectl logs coredns-74d6c5659f-24q2g -n kube-system
.:53
[INFO] plugin/reload: Running configuration MD5 = 438180e1b4b5c9e20262965fba0c62ae
CoreDNS-1.8.6
linux/amd64, go1.17.1, 13a9191
```
注意log日志中的报错信息。

### 检查 DNS 服务
如果所有CoreDNSPod 都运行良好并且日志中没有错误，下一步就是检查 DNS 服务。

```shell
# 使用kubectl get svc 验证 DNS 服务是否启动正常
root@node1:~/ephemeral-containers# kubectl get svc -A -l k8s-app=kube-dns
NAMESPACE     NAME       TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)                  AGE
kube-system   coredns    ClusterIP   10.233.0.3    <none>        53/UDP,53/TCP,9153/TCP   81d
kube-system   kube-dns   ClusterIP   10.233.0.10   <none>        53/UDP,53/TCP,9153/TCP   76d

#检查 DNS 服务endpoints是否正常暴露
root@node1:~/ephemeral-containers# kubectl get ep -A -l k8s-app=kube-dns
NAMESPACE     NAME       ENDPOINTS                                                        AGE
kube-system   coredns    10.233.64.148:53,10.233.65.103:53,10.233.64.148:53 + 3 more...   81d
kube-system   kube-dns   10.233.64.148:53,10.233.65.103:53,10.233.64.148:53 + 3 more...   76d
```
### 检查 CoreDNS 角色权限
CoreDNS RBAC的角色必须能List `service`和`endpoints`或者`endpointslices`相关资源的权限。
要检查权限，您可以获得当前的 ClusterRole `system:coredns`：

```shell
root@node1:~/ephemeral-containers# kubectl describe clusterrole system:coredns -n kube-system
Name:         system:coredns
Labels:       addonmanager.kubernetes.io/mode=Reconcile
              kubernetes.io/bootstrapping=rbac-defaults
Annotations:  <none>
PolicyRule:
  Resources                        Non-Resource URLs  Resource Names  Verbs
  ---------                        -----------------  --------------  -----
  nodes                            []                 []              [get]
  endpoints                        []                 []              [list watch]
  namespaces                       []                 []              [list watch]
  pods                             []                 []              [list watch]
  services                         []                 []              [list watch]
  endpointslices.discovery.k8s.io  []                 []              [list watch]
```
检查以上角色的权限，如果缺少任何权限，您可以编辑`ClusterRole`以添加缺少的权限：
```shell
kubectl edit clusterrole system:coredns -n kube-system
```

## CoreDNS性能优化


### 合理控制 CoreDNS 副本数
考虑以下几种方式:

1. 根据集群规模预估 coredns 需要的副本数，直接调整 coredns `deployment `的副本数:
```
kubectl -n kube-system scale --replicas=10 deployment/coredns
```
2. 为 coredns 定义 HPA 自动扩缩容。
安装 `cluster-proportional-autoscaler` 以实现更精确的扩缩容(推荐)。

### 禁用 ipv6
如果 K8S 节点没有禁用 IPV6 的话，容器内进程请求 coredns 时的默认行为是同时发起双栈 IPV4 和 IPV6 解析，而通常我们只需要用到 IPV4，当容器请求某个域名时，coredns 解析不到 IPV6 记录，就会 forward 到 upstream 去解析，如果到 upstream 需要经过较长时间(比如跨公网，跨机房专线)，就会拖慢整个解析流程的速度，业务层面就会感知 DNS 解析慢。

CoreDNS 有一个 [template](https://coredns.io/plugins/template/) 的插件，可以用它来禁用 IPV6 的解析，只需要给 CoreDNS 加上如下的配置:
```
template ANY AAAA {
    rcode NXDOMAIN
}
```
>这个配置的含义是：给所有 IPV6 的解析请求都响应空记录，即无此域名的 IPV6 记录。

### 优化 ndots
默认情况下，Kubernetes 集群中的域名解析往往需要经过多次请求才能解析到。查看 pod 内 的 /etc/resolv.conf 可以知道 ndots 选项默认为 5:
```shell
/ # cat /etc/resolv.conf
search default.svc.cluster.local svc.cluster.local cluster.local
nameserver 169.254.25.10
options ndots:5
```
`/etc/resolv.conf`包含3条配置信息
- nameserver IP是Kubernetes的CoreDNS的IP
- 指定了 3 个本地search域
- ndots:5选项

**什么是FQDN(完全限定名称)**

FQDN(完全限定名称)是不会对其在本地搜索域进行解析的名称，并且在解析期间该名称将被视为绝对名称。按照惯例，如果以句号 `(.) `结尾，则 CoreDNS认为名称是FQDN(完全限定名称)的，否则是非完全限定的。
比如：google.com.是FQDN(完全限定名称)的绝对名称，google.com不是绝对名称。

**非FQDN(完全限定名称)解析是如何执行的**

当应用程序连接到远程主机时需要解析主机名称，通常会通过系统调用执行 DNS 解析，例如getaddrinfo(). 如果名称不是完全限定的（不以`.`结尾.），CoreDNS会首先尝试将名称解析为绝对名称，还是首先通过本地搜索域？这取决于ndots选项。

`From the resolv.conf man:`
```shell
ndots:n

sets a threshold for the number of dots which must appear in a name before an initial absolute query will be made. The default for n is 1, meaning that if there are any dots in a name, the name will be tried first as an absolute name before any search list elements are appended to it.
```

这意味着如果ndots设置为5并且主机名称中包含少于 5 个点`.`，则系统调用将尝试首先依次遍历所有本地搜索域来解析它，如果没有成功 最后只会将其解析为绝对名称.

举个示例：
举个例子，在`default命名空间`查询 `kubernetes.default` 这个 service:

1. 域名中有1个.，小于 5，尝试拼接上第一个search域进行查询，即`kubernetes.default.default.svc.cluster.local`，查不到该域名。
2. 继续尝试`kubernetes.default.svc.cluster.local`，查不到该域名。
3. 继续尝试 `kubernetes.default.cluster.local`，查询成功，返回响应的 ClusterIP。
可以看到一个简单的 service 域名解析需要经过 3 轮解析才能成功，集群中充斥着大量无用的 DNS 请求。
鉴于以上情况：

业务发请求时尽量将 service 域名拼完整，这样就不会经过 search 拼接造成大量多余的 DNS 请求。

我们可以设置较小的 ndots，在 Pod 的 `dnsConfig` 中可以设置:
允许通过dnsConfigpod 属性对 Pod 的 DNS 设置进行更多控制。
除其他外，它允许自定义ndots特定 pod 的值，即：

```yaml
apiVersion: v1
kind: Pod
metadata:
  namespace: default
  name: dns-example
spec:
  containers:
    - name: test
      image: nginx
  dnsConfig:
    options:
      - name: ndots
        value: "2"
```

### 启用 autopath 插件
启用 CoreDNS 的 autopath 插件可以避免每次域名解析经过多次请求才能解析到，原理是 CoreDNS 智能识别拼接过 search 的 DNS 解析，直接响应 CNAME 并附上相应的 ClusterIP，一步到位，可以极大减少集群内 DNS 请求数量。
启用方法是修改 CoreDNS 配置:
```
kubectl -n kube-system edit configmap coredns
```

```
.:53 {
    errors
    health {
        lameduck 5s
    }
    ready
    kubernetes cluster.local in-addr.arpa ip6.arpa {
      pods verified
      fallthrough in-addr.arpa ip6.arpa
    }
    prometheus :9153
    forward . /etc/resolv.conf {
      prefer_udp
    }
    cache 30
    autopath @kubernetes

    loop
    reload
    loadbalance
}
```
- `pods verified`
- `autopath @kubernetes`

需要注意的是，启用 autopath 后，由于 coredns 需要 watch 所有的 pod，会增加 coredns 的内存消耗，根据情况适当调节 coredns 的 memory request 和 limit。

### 部署 NodeLocal DNSCache

参考 k8s 官方文档 [Using NodeLocal DNSCache in Kubernetes clusters](https://kubernetes.io/docs/tasks/administer-cluster/nodelocaldns/)


### 升级系统内核避免conntrack race condition

建议升级至Kernnel 5.9 以上,内核netfilter模块nf_conntrack用一个哈希表记录已建立的连接，包括其他机器到本机、本机到其他机器、本机内部的连接，如果连接进来比释放得快，把这个哈希表塞满了，新连接的数据包会被drop掉,确认这部分的配置不会成为瓶颈,如果云服务厂商有提供安全组、防火墙等功能，可以直接使用外部防火墙，此时可以直接撤掉机器内部的防火墙，也就没有这个问题了。亦或者如果不需要使用到涉及到状态链路相关或者NAT相关的应用，也可以直接设置不加载这个模块的。


### IPVS 参数优化

```
net.ipv4.vs.conn_reuse_mode = 1
net.ipv4.vs.conntrack = 1
net.ipv4.vs.expire_nodest_conn = 1
```
 sys.net.ipv4.vs.conn_reuse_mode值为 0，ipvs 中，系统使用conntrack跟踪连接，conntrack使用三元组（即源IP、源端口、协议）定位具体连接，当客户端产生大量短连接时，其本身可能与之前的端口复用，而 TCP 在conntrack中的超时时间达 900s，也就是说conntrack中记录的连接可能存在很长时间，客户端过来的新连接极有可能复用老连接，此时已经 TIME_WAIT 的老连接无法响应请求，从而导致超时。可通过设置sys.net.ipv4.vs.conn_reuse_mode的值为 1 来解决

内核参数net.ipv4.vs.expire_nodest_conn，用于控制连接的rs不可用时的处理。在开启时，如果后端rs不可用，会立即结束掉该连接，使客户端重新发起新的连接请求；否则将数据包silently drop，也就是DROP掉数据包但不结束连接，等待客户端的重试


## 参考文档

[https://mydream.ink/posts/cloud-native/kubernetes/record-a-kubernetes-service-exception-caused-by-ipvs/](https://mydream.ink/posts/cloud-native/kubernetes/record-a-kubernetes-service-exception-caused-by-ipvs/)

[https://github.com/kubernetes/kubernetes/issues/81775](https://github.com/kubernetes/kubernetes/issues/81775)

[https://imroc.cc/post/202105/ipvs-conn-reuse-mode/](https://imroc.cc/post/202105/ipvs-conn-reuse-mode/)
