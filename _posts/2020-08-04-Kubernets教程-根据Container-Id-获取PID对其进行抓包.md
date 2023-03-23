---
layout:     post
title:      Kubernets教程：根据Container ID 获取PID对其进行抓包
subtitle:   深入到底层去进行网络抓包分析报文并进行故障排查
date:       2020-08-04
author:     J
catalog: true
tags:
    - kubernetes	
---

  在管理 `Kubernetes` 集群的过程中，我们经常会遇到这样一种情况：碰到一些复杂的故障问题需要我们深入底层网络层进行抓包分析报文的时候，需要我们对进出容器的网络报文进行抓取，如果熟悉容器的原理的话，就会知道容器其实就是一些特殊的进程，对容器进行网络抓包就是对进程进行抓包，知道这个之后就可以在容器所在的宿主机上进行`tcpdump`抓包分析报文

> 以下就以`coredns`这个Pod作为例子进行抓包分析`DNS`报文分析

```shell
root@node2:~# kubectl get pod -o wide -n kube-system -l k8s-app=kube-dns
NAME                       READY   STATUS    RESTARTS   AGE    IP             NODE    NOMINATED NODE   READINESS GATES
coredns-85578f88b8-n2t6g   1/1     Running   0          5d1h   172.20.1.195   node2   <none>           <none>
coredns-85578f88b8-vkpts   1/1     Running   0          5d1h   172.20.2.93    node3   <none>           <none>
```

可以看到`CoreDNS`的Pod目前被调度到`node2`和`node3`上,那就以`node2`上的为例

### 1. 获取Container ID

```shell
root@node2:~# kubectl -n kube-system describe pod coredns-85578f88b8-n2t6g | grep "Container ID"
    Container ID:  containerd://72651dc70ee9e7d95f2784451e4453cbb9dbd4e875bcf94b191bfadeb147eac4
```

这里是用`containerd`作为容器的`Runtime`

### 2. 获取进程PID

```shell
root@node2:~# ps -ef |grep -v grep |grep 72651dc70ee9e7d95f2784451e4453cbb9dbd4e875bcf94b191bfadeb147eac4 
root       5162   1341  0 Jul30 ?        00:00:12 /usr/bin/containerd-shim -namespace k8s.io -workdir /var/lib/containerd/io.containerd.runtime.v1.linux/k8s.io/72651dc70ee9e7d95f2784451e4453cbb9dbd4e875bcf94b191bfadeb147eac4 -address /run/containerd/containerd.sock -containerd-binary /usr/bin/containerd
```

这里我们可以看到这个container对应在node2上的进程号是`5162`

### 3. 创建一个`busybox`的Pod做域名解析请求`CoreDNS`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nslookup-busybox
  namespace: default
  labels:
    app: busybox
spec:
  hostNetwork: true
  containers:
  - image: busybox:1.28.3
    command:
      - sleep
      - "3600"
    imagePullPolicy: IfNotPresent
    name: busybox
  restartPolicy: Always
  dnsPolicy: ClusterFirstWithHostNet
```

> *busybox*: latest镜像在进行*nslookup*会报错，慎用
>
> https://github.com/docker-library/busybox/issues/48

```shell
root@node1:~/k8s_manifests/busybox# kubectl apply -f busybox.yaml 
pod/nslookup-busybox created
root@node1:~/k8s_manifests/busybox# kubectl get pod -n default
NAME               READY   STATUS              RESTARTS   AGE
nslookup-busybox   0/1     ContainerCreating   0          12s
root@node1:~/k8s_manifests/busybox# kubectl get pod -n default
NAME               READY   STATUS    RESTARTS   AGE
nslookup-busybox   1/1     Running   0          24s
```

进入Pod进行`nslookup` 指定DNS做域名解析操作

```shell
root@node1:~/k8s_manifests/busybox# kubectl -n default exec -it nslookup-busybox sh
/ # nslookup google.com
Server:    169.254.25.10
Address 1: 169.254.25.10

Name:      google.com
Address 1: 2404:6800:4005:803::200e hkg07s22-in-x0e.1e100.net
Address 2: 216.58.200.14 hkg12s11-in-f14.1e100.net
```

> 由于当前集群启用了`NodeLocalDNS`的，默认会先去查询缓存，所以就需要我们指定`CoreDNS`的`svc ip`

```shell
root@node1:~/k8s_manifests/busybox# kubectl -n default exec -it nslookup-busybox sh
/ # nslookup google.com 10.244.0.3
Server:    10.244.0.3
Address 1: 10.244.0.3 coredns.kube-system.svc.cluster.local

Name:      google.com
Address 1: 2404:6800:4005:803::200e hkg07s22-in-x0e.1e100.net
Address 2: 216.58.200.14 hkg12s11-in-f14.1e100.net
```

### 4. 进`CoreDNS`的`namespace`进行`tcpdump`抓包

```shell
root@node2:~# nsenter -t 5162 -n tcpdump -i ens33 'udp and port 53' -vvv
tcpdump: listening on ens33, link-type EN10MB (Ethernet), capture size 262144 bytes
09:07:26.431305 IP (tos 0x0, ttl 62, id 2297, offset 0, flags [DF], proto UDP (17), length 56)
    node2.cluster.local.47373 > phicomm.me.domain: [bad udp cksum 0x8592 -> 0xad2f!] 3+ AAAA? google.com. (28)
09:07:26.432519 IP (tos 0x0, ttl 64, id 22616, offset 0, flags [DF], proto UDP (17), length 81)
    node2.cluster.local.52775 > phicomm.me.domain: [bad udp cksum 0x85ab -> 0xbca1!] 14745+ [1au] PTR? 1.2.168.192.in-addr.arpa. ar: . OPT UDPsize=512 (53)
09:07:26.432747 IP (tos 0x0, ttl 64, id 22617, offset 0, flags [DF], proto UDP (17), length 81)
    node2.cluster.local.44958 > phicomm.me.domain: [bad udp cksum 0x85ab -> 0x9699!] 32298+ [1au] PTR? 1.2.168.192.in-addr.arpa. ar: . OPT UDPsize=512 (53)
09:07:26.434046 IP (tos 0x0, ttl 64, id 0, offset 0, flags [DF], proto UDP (17), length 84)
    phicomm.me.domain > node2.cluster.local.47373: [udp sum ok] 3 q: AAAA? google.com. 1/0/0 google.com. [19h41m23s] AAAA 2404:6800:4005:803::200e (56)
09:07:26.434076 IP (tos 0x0, ttl 64, id 0, offset 0, flags [DF], proto UDP (17), length 105)
    phicomm.me.domain > node2.cluster.local.52775: [udp sum ok] 14745* q: PTR? 1.2.168.192.in-addr.arpa. 1/0/1 1.2.168.192.in-addr.arpa. [0s] PTR phicomm.me. ar: . OPT UDPsize=4096 (77)
09:07:26.434086 IP (tos 0x0, ttl 64, id 0, offset 0, flags [DF], proto UDP (17), length 105)
    phicomm.me.domain > node2.cluster.local.44958: [udp sum ok] 32298* q: PTR? 1.2.168.192.in-addr.arpa. 1/0/1 1.2.168.192.in-addr.arpa. [0s] PTR phicomm.me. ar: . OPT UDPsize=4096 (77)
09:07:26.439027 IP (tos 0x0, ttl 62, id 2299, offset 0, flags [DF], proto UDP (17), length 56)
    node2.cluster.local.47373 > phicomm.me.domain: [bad udp cksum 0x8592 -> 0xad49!] 4+ A? google.com. (28)
09:07:26.439614 IP (tos 0x0, ttl 64, id 0, offset 0, flags [DF], proto UDP (17), length 72)
    phicomm.me.domain > node2.cluster.local.47373: [udp sum ok] 4 q: A? google.com. 1/0/0 google.com. [19h41m25s] A 216.58.200.14 (44)
09:07:26.444809 IP (tos 0x0, ttl 62, id 2300, offset 0, flags [DF], proto UDP (17), length 118)
    node2.cluster.local.47373 > phicomm.me.domain: [bad udp cksum 0x85d0 -> 0x4da3!] 5+ PTR? e.0.0.2.0.0.0.0.0.0.0.0.0.0.0.0.3.0.8.0.5.0.0.4.0.0.8.6.4.0.4.2.ip6.arpa. (90)
09:07:26.446774 IP (tos 0x0, ttl 64, id 0, offset 0, flags [DF], proto UDP (17), length 157)
    phicomm.me.domain > node2.cluster.local.47373: [udp sum ok] 5 q: PTR? e.0.0.2.0.0.0.0.0.0.0.0.0.0.0.0.3.0.8.0.5.0.0.4.0.0.8.6.4.0.4.2.ip6.arpa. 1/0/0 e.0.0.2.0.0.0.0.0.0.0.0.0.0.0.0.3.0.8.0.5.0.0.4.0.0.8.6.4.0.4.2.ip6.arpa. [23h41m24s] PTR hkg07s22-in-x0e.1e100.net. (129)
09:07:26.454879 IP (tos 0x0, ttl 62, id 20349, offset 0, flags [DF], proto UDP (17), length 72)
    node3.cluster.local.52778 > phicomm.me.domain: [udp sum ok] 6+ PTR? 14.200.58.216.in-addr.arpa. (44)
09:07:26.454884 IP (tos 0x0, ttl 64, id 0, offset 0, flags [DF], proto UDP (17), length 111)
    phicomm.me.domain > node3.cluster.local.52778: [udp sum ok] 6 q: PTR? 14.200.58.216.in-addr.arpa. 1/0/0 14.200.58.216.in-addr.arpa. [23h41m25s] PTR hkg12s11-in-f14.1e100.net. (83)
09:07:55.564337 IP (tos 0x0, ttl 64, id 20620, offset 0, flags [none], proto UDP (17), length 62)
```

> 还可以直接输出到cap格式文件，在**Wireshark**中进行分析过滤掉其他报文

