---
layout:     post
title:      nf_conntrack races 引发的DNS lookup timeouts问题
subtitle:   Alpine gnu c库和musl libc库都并行执行A和AAAA DNS查找引出的问题
date:       2020-04-07
author:     J
catalog: true
tags:
    - kubernetes
---

之前一篇关于Alpine的基础镜像使用的注意事项中有一个`DNS`解析异常，这个跟是否使用`musl libc`库或者是`gnu libc`库没有关系

##  背景

在NAT环境中，当多个数据包从相同的起点同时发送到相同的目的地时，nf_conntrack中存在许多竞争，其中没有预先确定的conntrack条目。

这在具有kubernetes工作负载的用户中最常见，其中dns 请求是从同一套接字同时从不同线程发送的。一个请求是A记录，另一个是AAAA。发生conntrack竞赛，这导致一个请求被丢弃，从而导致5秒dns超时。

这个问题并非特定于kubernetes，因为任何从同一线程同时从不同线程同时发送UDP数据包的多线程进程都会受到影响。

请注意，由于UDP是无连接的，因此在调用connect（）时不会发送任何数据包，因此在第一次发送数据包之前不会创建conntrack条目。

从不同线程上的同一套接字同时发送两个UDP数据包的情况下，存在以下可能的竞争：

- 两个数据包均未找到确认的conntrack条目。这将导致使用相同的元组创建两个conntrack条目。

- 与1）相同，但是在另一个调用get_unique_tuple（）之前，已为其中一个数据包确认了conntrack条目。另一个数据包在源端口更改的情况下获得了不同的回复元组。

比赛的结果是相同的，当需要使用__nf_conntrack_ confirm（）确认conntrack条目时，将丢弃其中一个数据包。

> GNU C库和musl libc都并行执行A和AAAA DNS查找。由于竞争，内核可能会丢弃其中一个UDP数据包，因此客户端通常会在5秒的超时后尝试重新发送它。

### 解决方案一: 使用 TCP 发送 DNS 请求

如果使用 TCP 发 DNS 请求，connect 时就会发包建立连接并插入 conntrack 表项，而后并发的 A 和 AAAA 记录的请求在 send 时都使用 connect 建立好的这个 fd，由于 connect 时 conntrack 表项已经建立，所以 send 时不会再建立，也就不存在并发创建 conntrack 表项，避免了冲突。

`resolv.conf` 可以加 `options use-vc` 强制 glibc 使用 TCP 协议发送 DNS query。下面是这个 `man resolv.conf`中关于这个选项的说明:

```bash
use-vc (since glibc 2.14)
                     Sets RES_USEVC in _res.options.  This option forces the
                     use of TCP for DNS resolutions.
```

### 解决方案二: 避免相同五元组 DNS 请求的并发

`resolv.conf` 还有另外两个相关的参数：

- single-request-reopen (since glibc 2.9): A 和 AAAA 请求使用不同的 socket 来发送，这样它们的源 Port 就不同，五元组也就不同，避免了使用同一个 conntrack 表项。
- single-request (since glibc 2.10): A 和 AAAA 请求改成串行，没有并发，从而也避免了冲突。

### 解决方案三: 使用本地 DNS 缓存

仔细观察可以看到前面两种方案是 glibc 支持的，而基于 alpine 的镜像底层库是 musl libc 不是 glibc，所以即使加了这些 options 也没用，这种情况可以考虑使用本地 DNS 缓存来解决，容器的 DNS 请求都发往本地的 DNS 缓存服务(dnsmasq, nscd, coredns等)，不需要走 DNAT，也不会发生 conntrack 冲突。另外还有个好处，就是避免 DNS 服务成为性能瓶颈。

使用本地DNS缓存有两种方式：

- 每个容器自带一个 DNS 缓存服务
- 每个节点运行一个 DNS 缓存服务，所有容器都把本节点的 DNS 缓存作为自己的 nameserver

### 解决方案四：升级内核

升级至stable release （4.9.163, 4.14.106, 4.19.29, 4.20.16）以上

ubuntu 18.04（Bionic）修复该[bug](https://bugs.launchpad.net/ubuntu/+source/linux/+bug/1836816)

### 总结：

以上四种解决方案可以根据具体的生产环境来做相应的调整，这边推荐使用**本地DNS缓存**和**增加CoreDNS的副本**来缓解这个Conntrack竞争的问题



参考文档：

https://medium.com/@danielmller_75561/performance-issues-with-rds-aurora-on-eks-due-to-coredns-defaults-5fb2166366c9

https://bugs.launchpad.net/ubuntu/+source/linux/+bug/1836816

https://www.weave.works/blog/racy-conntrack-and-dns-lookup-timeouts

https://blog.quentin-machu.fr/2018/06/24/5-15s-dns-lookups-on-kubernetes/

https://tech.xing.com/a-reason-for-unexplained-connection-timeouts-on-kubernetes-docker-abd041cf7e02

https://imroc.io/posts/troubleshooting-with-kubernetes-network/
