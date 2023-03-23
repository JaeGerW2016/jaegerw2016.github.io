---
layout:     post
title:      Kubernetes启用Node Local DNS Cache
subtitle:   解决nf_conntrack race的问题
date:       2020-04-27
author:     J
catalog: true
tags:
    - kubernetes
---



[Node Local DNS Cache](https://github.com/kubernetes/enhancements/blob/master/keps/sig-network/20190424-NodeLocalDNS-beta-proposal.md)。这是一款非常酷的软件，它通过将大多数响应缓存在节点本地DNS上来帮助DNS加载，并解决了`Linux conntrack`争用问题，这会导致某些DNS请求出现5s的间歇性延迟。你可以阅读更多有关contrack问题[5S的DNS间歇延迟发布](https://github.com/kubernetes/kubernetes/issues/56903)和[活跃的跟踪连接和DNS查找超时](https://www.weave.works/blog/racy-conntrack-and-dns-lookup-timeouts)

### DNS在Kubernetes中的典型工作方式

默认情况下，Kubernetes中的Pod将请求发送到*kube-dns*服务。在内部，这是通过将pod的`/etc/resolv.conf` `nameserver`字段设置为*kube-dns*服务`svc IP`（`10.244.0.3`默认情况下）来完成的。 *kube-proxy*管理这些服务IP，并通过*iptables*或*IPVS*技术将它们转换为*kube-dns*的`endpoint IP`。使用`Node Local DNS Cache`，pod可以直接请求`Node Local DNS Cache`的`svc IP`，而无需任何连接跟踪。如果请求在pod的缓存中，那么我们可以直接返回响应。否则，`Node Local DNS Cache`会通过TCP请求上游*kube-dns*的`svc IP`，从而避免*conntrack*错误。

### Node Local DNS Cache 部署方式

部署`Node Local DNS Cache`，您将必须选择本地链接地址段,。本地链接地址段是IP地址的特殊类别，范围为*169.254.0.1*至*169.254.255.254*。定义在 [RFC3927](http://tools.ietf.org/html/rfc3927) 。重要的是，路由器不转发带有本地链接地址段的数据包，因为不能保证该范围在其网段之外是唯一的。示例部署使用`169.254.25.10`默认的本地链接地址。

集群正在`ipvs`模式下运行kube-proxy，而*kube-dns*的`svc IP`是`10.244.0.3` 并且使用`169.254.25.10`作为本地链接地址。

首先，因为部署`Node Local DNS Cache`是通过`Daemonset`的形式，并且是配置为`hostNetwork: true`然后在每个主机本地创建一个`nodelocaldns`的网络接口

```shell
4: kube-ipvs0: <BROADCAST,NOARP> mtu 1500 qdisc noop state DOWN group default 
    link/ether fe:43:bf:53:02:89 brd ff:ff:ff:ff:ff:ff
...
    inet 10.244.0.3/32 brd 10.244.0.3 scope global kube-ipvs0
       valid_lft forever preferred_lft forever
...
12: nodelocaldns: <BROADCAST,NOARP> mtu 1500 qdisc noop state DOWN group default 
    link/ether e2:13:08:c8:65:24 brd ff:ff:ff:ff:ff:ff
    inet 169.254.25.10/32 brd 169.254.25.10 scope global nodelocaldns
       valid_lft forever preferred_lft forever
...
```

节点本地DNS将同步`iptables`规则。你可以通过执行看到这些规则`iptables-save`，寻找`169.254.25.10`IP相关的规则。这是我在一个节点上看到的

```shell
*raw
...
-A PREROUTING -d 169.254.25.10/32 -p udp -m udp --dport 53 -j NOTRACK
-A PREROUTING -d 169.254.25.10/32 -p tcp -m tcp --dport 53 -j NOTRACK
...
-A OUTPUT -s 169.254.25.10/32 -p tcp -m tcp --sport 8080 -j NOTRACK
-A OUTPUT -d 169.254.25.10/32 -p tcp -m tcp --dport 8080 -j NOTRACK
-A OUTPUT -d 169.254.25.10/32 -p udp -m udp --dport 53 -j NOTRACK
-A OUTPUT -d 169.254.25.10/32 -p tcp -m tcp --dport 53 -j NOTRACK
-A OUTPUT -s 169.254.25.10/32 -p udp -m udp --sport 53 -j NOTRACK
-A OUTPUT -s 169.254.25.10/32 -p tcp -m tcp --sport 53 -j NOTRACK
...

*filter
...
-A INPUT -d 169.254.25.10/32 -p udp -m udp --dport 53 -j ACCEPT
-A INPUT -d 169.254.25.10/32 -p tcp -m tcp --dport 53 -j ACCEPT
...
-A OUTPUT -s 169.254.25.10/32 -p udp -m udp --sport 53 -j ACCEPT
-A OUTPUT -s 169.254.25.10/32 -p tcp -m tcp --sport 53 -j ACCEPT
...

```

ipvs转发规则

```shell
root@node1:~/k8s_manifests/busybox# kubectl describe ep coredns -n kube-system
Name:         coredns
Namespace:    kube-system
Labels:       addonmanager.kubernetes.io/mode=Reconcile
              k8s-app=kube-dns
              kubernetes.io/name=coredns
Annotations:  <none>
Subsets:
  Addresses:          172.20.0.228,172.20.1.223
  NotReadyAddresses:  <none>
  Ports:
    Name     Port  Protocol
    ----     ----  --------
    dns      53    UDP
    dns-tcp  53    TCP
    metrics  9153  TCP

Events:  <none>

# ipvsadm -L
UDP  node1:domain rr
  -> 172.20.0.228:domain          Masq    1      0          0         
  -> 172.20.1.223:domain          Masq    1      0          0 
TCP  node1:9153 rr
  -> 172.20.0.228:9153            Masq    1      0          0         
  -> 172.20.1.223:9153            Masq    1      0          0
```

任何*kube-proxy*规则之前添加这些规则，以便首先对其进行评估。`-j NOTRACK`使发送到本地`169.254.25.10`地址的TCP和UDP数据包被conntrack取消跟踪。这就是节点本地DNS缓存避免conntrack错误并接管数据包以发送到*kube-dns* IP的方式

### CoreDNS配置

```yaml
# kubectl get configmap nodelocaldns -n kube-system -o yaml
...
Corefile: |
    cluster.local:53 {
        errors
        cache {
            success 9984 30
            denial 9984 5
        }
        reload
        loop
        bind 169.254.25.10
        forward . 10.244.0.3 {
            force_tcp
        }
        prometheus :9253
        health 169.254.25.10:9254
    }
    in-addr.arpa:53 {
        errors
        cache 30
        reload
        loop
        bind 169.254.25.10
        forward . 10.244.0.3 {
            force_tcp
        }
        prometheus :9253
    }
    ip6.arpa:53 {
        errors
        cache 30
        reload
        loop
        bind 169.254.25.10
        forward . 10.244.0.3 {
            force_tcp
        }
        prometheus :9253
    }
    .:53 {
        errors
        cache 30
        reload
        loop
        bind 169.254.25.10
        forward . 10.244.154.167 {
            force_tcp
        }
        prometheus :9253
    }
...
```

这部分配置中`cluster.local`区域，负责处理kubernetes集群内部的DNS解析，在此区域中，可以看到配置了

缓存，成功和拒绝的记录限制都为9984条，成功有效生存时间30s，拒绝的有效生存时间为5s 。如果请求未命中缓存，就强制使用TCP（避免conntrack race错误）将请求发送到`10.244.0.3`这个`__PILLAR__CLUSTER__DNS`地址。

还有一个`.:53`区域，该区域处理如果解析请求不是针对Kubernetes内部运行的服务的情况。我们缓存请求并转发到`__PILLAR__UPSTREAM__SERVERS`上游DNS服务器。节点本地DNS `__PILLAR__UPSTREAM__SERVERS`从*kube-dns* configmap 查找值。该示例部署未设置它，因此默认为`/etc/resolv.conf`。请注意，节点本地DNS使用`dnsPolicy: Default`，这`/etc/resolv.conf`与节点上的相同。
> 为了避免 coredns的 hosts和rewrite配置不生效，得在nodelocaldns里 forward . /etc/resolv.conf 修改成 forward . __PILLAR__UPSTREAM__SERVERS

```yaml
piVersion: v1
kind: Service
metadata:
  name: kube-dns-upstream
  namespace: kube-system
  labels:
    k8s-app: kube-dns
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
    kubernetes.io/name: "KubeDNSUpstream"
spec:
  ports:
  - name: dns
    port: 53
    protocol: UDP
    targetPort: 53
  - name: dns-tcp
    port: 53
    protocol: TCP
    targetPort: 53
  selector:
    k8s-app: kube-dns
```
获取`kube-dns-upstream`的service IP
```shell
kubectl get svc -n kube-system | grep kube-dns-upstream
```
### 高可用性

这部分高可用性取决于`Node Local DNS Cache`的pod对于信号的处理，如果Pod是正常终止，Pod将删除nodelocaldns这个虚拟网络地址和iptables的规则，然后将请求的流量从自身切换到集群中的*kube-dns*的Pod上，不影响正常的DNS解析，如果是强行OOM被kill掉的话，则不会删除iptables规则，这样的话将影响正常的DNS解析。

为了避免不必要的麻烦，默认的Daemonset没有设置memory/CPU限制，以避免Pod因内存不足或者CPU节流被非正常kill掉，此外如果pod的annotation被标记为`system-node-critical`使得它们几乎可以保证在节点资源不足时首先进行调度，而不太可能被驱逐。

同时为了 安全起见，建议将Daemonset的更新策略更改为OnDelete 通过手动额外删除Node Local DNS pod 来进行维护/升级

### 参考文档

https://povilasv.me/kubernetes-node-local-dns-cache/#

https://github.com/kubernetes/enhancements/blob/master/keps/sig-network/20190424-NodeLocalDNS-beta-proposal.md
