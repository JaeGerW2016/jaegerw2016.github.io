---
layout: mypost
title: DNS Timeout Storm caused by sudo hostname lookups
categories: [Linux CoreDNS Kubernetes]  
---

## 背景

在日常使用 Linux 的时候，我们经常会注意到即便是简单的命令加上 sudo，执行时也会出现卡顿几秒钟的现象。在一些复杂环境（如 Kubernetes/EKS 大规模集群）中，这种现象甚至可能演变成大规模 DNS 超时问题。要理解原因，需要先弄清楚 sudo 与 DNS 的关系。

### sudo 背后的主机名解析

当你运行 sudo 时，它并不仅仅是提权执行命令，还会调用 sudoers 插件来完成一系列校验：

确认调用者身份
检查 /etc/sudoers 配置规则
获取当前主机名，调用 gethostname()
尝试将主机名解析为完整限定域名（FQDN），调用 getaddrinfo()
最后这一步就是关键 —— 一旦主机名未能在 /etc/hosts 中直接解析，操作系统就会尝试通过 DNS 查询来完成解析。

使用`ltrace/strace`追踪进行troubleshooting....

```shell
root@debian:~# ltrace -f -e 'getaddrinfo*+gethostby*+gethostname*' sudo -n uptime

[pid 3373] libsudo_util.so.0->gethostname("debian", 65)                = 0
[pid 3373] sudoers.so->getaddrinfo("debian", nil, 0x7ffead9a7b40, 0x7ffead9a7b38) = 0
[pid 3375] --- Called exec() ---
[pid 3375] +++ exited (status 0) +++
[pid 3374] --- SIGCHLD (Child exited) ---

22:47:18 up 3 days,  2:12,  4 users,  load average: 0.00, 0.00, 0.00
[pid 3374] +++ exited (status 1) +++
pid 3373] --- SIGCHLD (Child exited) ---
[pid 3373] +++ exited (status 0) +++

```
`sudoers.so->getaddrinfo` 这行日志输出清楚表明是`sudo`命令本身发起的DNS请求。

### 在容器/云环境中的放大效应

在传统单机环境中，即使发生 DNS 查询问题，影响范围相对较小。但在容器和云环境（例如 AWS EKS）中，这种行为被放大：

Kubernetes 的 resolv.conf 配置带有搜索域和 ndots 参数

ndots:5 意味着任何不含 5 个点的域名都会被追加多个搜索域进行解析。
一个简单的 hostname 解析就可能触发 多次 DNS 查询。
每次查询还会包含 A 和 AAAA 记录，查询量翻倍。
集群 DNS（CoreDNS）作为中心化服务

sudo 带来的大量主机名解析请求，都会转发给 CoreDNS。
Cache 未命中时还会继续转发至 VPC 内置的 Route53 resolver。
AWS 的 ENI PPS 限制

每个 ENI 对 link-local 地址的请求限制为 1024 packets per second。
当请求量过大时，部分 UDP 包会被直接丢弃，表现为 i/o timeout，导致应用或 CI/CD 任务出现 DNS 解析失败。

### 案例：sudo 导致的 DNS 风暴

[My process to debug DNS timeouts in a large EKS cluster](https://cep.dev/posts/eks-dns-timeouts-sudo-hostname-lookups/)

在以上的博客内容中，可以看到在一次排查中，发现节点上大量 DNS 请求都来自一些“怪异的”域名，比如：

ip-10-0-1-10.us-east-2.compute.internal.cluster.local ip-10-0-1-10.us-east-2.compute.internal.svc.cluster.local ...
这些查询实际上来自容器内的 sudo 调用 hostname 解析，由于 Kubernetes 搜索域 + ndots 配置，导致一次解析衍生出 5~10 次 DNS 请求。规模足够大的时候，就突破了 VPC resolver 的 PPS 限制，给整个集群带来不稳定性。


### 如何解决

- 尽量避免使用Alpine 默认使用musl lib的基础镜像,musl lib不支持 nsswitch.conf，实现了 更简单的解析逻辑。lookup 是固定流程，不能通过 NSS 配置灵活切换。

- 为 hostname 提前写入 /etc/hosts  保证 getaddrinfo() 直接命中，无需 DNS 请求。

- 优化 DNS 配置 考虑降低 ndots 的值，比如改成 ndots:2。 避免不必要的 DNS 请求。 监控与调优 CoreDNS 和 VPC DNS 解析

- 增加 CoreDNS 副本，开启NodeLocalDNS，减少 DNS 上游压力。

- 配置CoreDNS 开启auto-path插件避免不必要的DNS域名拼接请求

- 在 sudo 配置中禁用 FQDN 解析

```
echo 'Defaults !fqdn' | sudo tee /etc/sudoers.d/00-no-fqdn
sudo chmod 440 /etc/sudoers.d/00-no-fqdn
```
### From Sudo to VPC Resolver的Mermaid 流程图

![Mermaid 流程图](https://tc.z.wiki/autoupload/f/mwggrcggsT8p42qdtkcPalvVXXK6JfkDvwLsSDlD2H2yl5f0KlZfm6UsKj-HyTuv/20250918/PK5S/2560X3840/_Mermaid_Chart-2025-09-18-032101.png/webp)

### 总结

很多人没想到 —— 一个简单的 sudo 命令居然能引发集群级别的 DNS 异常。

其根源在于：

sudo 默认会做主机名解析
Kubernetes 的 ndots 和搜索域机制扩大了查询次数
云厂商的 link-local PPS 限制成为了瓶颈
最终形成了 DNS 超时甚至服务抖动。

所以在云原生环境里，合理配置 sudo、/etc/hosts 以及 resolv.conf，不仅仅是一个效率优化问题，而是直接关系到集群 DNS 的稳定性。


### 参考文档

[https://lists.debian.org/debian-user/2011/10/msg02014.html](https://lists.debian.org/debian-user/2011/10/msg02014.html)

[https://unix.stackexchange.com/questions/766131/why-sudo-delays-5s-to-10s-if-wifi-is-enabled-and-how-to-fix-it](https://unix.stackexchange.com/questions/766131/why-sudo-delays-5s-to-10s-if-wifi-is-enabled-and-how-to-fix-it)


[https://docs.aws.amazon.com/vpc/latest/userguide/AmazonDNS-concepts.html](https://docs.aws.amazon.com/vpc/latest/userguide/AmazonDNS-concepts.html)
