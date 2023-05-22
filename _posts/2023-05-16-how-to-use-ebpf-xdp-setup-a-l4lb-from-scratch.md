---
layout: mypost
title: 使用ebpf的XDP来创建一个简单L4负载均衡
categories: [ebpf,XDP]
---

### 什么是 eBPF XDP

eBPF是 Berkeley Packet Filter (BPF) 的扩展版本。它是在 Linux 内核中运行的抽象虚拟机 (VM)，很像 Java 虚拟机 (JVM)，可以在受控环境中运行应用程序。eBPF 可以在内核的沙箱内执行用户定义的程序——它通常用于使开发人员能够以确保最佳性能的方式在 Linux 中编写低级监控、跟踪或网络程序。

eXpress Data Path (XDP) 是一个框架，可以在 BPF 应用程序中执行高速数据包处理。为了能够更快地响应网络操作，XDP 会尽快运行 BPF 程序，通常是在网络接口收到数据包时立即运行。

![what is ebpf](https://ebpf.io/static/e293240ecccb9d506587571007c36739/f2674/overview.png)

### eBPF 中对 XDP（eXpress Data Path） 的需求
XDP 是一种允许开发人员将 eBPF 程序附加到低级挂钩的技术，低级挂钩由 Linux 内核中的网络设备驱动程序实现，以及在设备驱动程序之后运行的通用挂钩。

XDP 可用于在 eBPF 架构中实现高性能数据包处理，主要使用内核旁路。这大大减少了内核所需的开销，因为它不需要处理上下文切换、网络层处理、中断等。网络接口卡 (NIC) 的控制权转移到 eBPF 程序。如果您以更高的网络速度（10 Gbps 及以上）工作，这一点尤其重要。

但是，内核绕过方法有一些缺点：

- eBPF 程序必须编写自己的驱动程序。这给开发人员带来了额外的工作。
- XDP 程序在数据包被解析之前运行。这意味着 eBPF 程序必须直接实现它们完成工作所需的功能，而不依赖于内核。

这些限制产生了对 XDP 的需求。通过允许 eBPF 程序直接读写网络数据包数据，并在到达内核级别之前确定如何处理数据包，XDP 使得在 eBPF 中实现高性能网络变得更加容易。

### XDP（eXpress Data Path） 是如何工作的
XDP 程序可以直接附加到网络接口。每当在网络接口上接收到新数据包时，XDP 程序都会收到回调，并可以非常快速地对该数据包执行操作。

您可以使用以下模型将 XDP 程序连接到接口：

- `Generic XDP`  – 表示通用 XDP，用于给那些还没有原生支持 XDP 的驱动进行试验性测试。generic XDP hook 位于内核协议栈的主接收路径（main receive path）上，接受的是 skb 格式的包，但由于 这些 hook 位于 ingress 路径的很后面，因此与 native XDP 相比性能有明显下降。因此，xdpgeneric 大部分情况下只能用于试验目的，很少用于生产环境。
- `Native XDP` – 表示 native XDP（原生 XDP）, 意味着 BPF 程序直接在驱动的接收路 径上运行，理论上这是软件层最早可以处理包的位置（the earliest possible point）。这是常规/传统的 XDP 模式，需要驱动实现对 XDP 的支持，目前 Linux 内核中主流的 10G/40G 网卡都已经支持。


- `Offloaded XDP` – XDP 程序直接加载到 NIC 上，无需使用 CPU 即可执行。这需要网络接口设备的支持。一些智能网卡（例如支持 Netronome’s nfp 驱动的网卡）实现了 xdpoffload 模式 ，允许将整个 BPF/XDP 程序 offload 到硬件，因此程序在网卡收到包时就直接在网卡进行 处理。这提供了比 native XDP 更高的性能，虽然在这种模式中某些 BPF map 类型 和 BPF 辅助函数是不能用的。BPF 校验器检测到这种情况时会直 接报错，告诉用户哪些东西是不支持的。除了这些不支持的 BPF 特性之外，其他方面与 native XDP 都是一样的。

其中`Native XDP` 是默认模式。当谈到 XDP 时，通常会表示使用这种模式。

以下是 XDP 程序在连接到网络接口后可以对其接收到的数据包执行的一些`xdp_actions`：

- `XDP_DROP` – 顾名思义，它将在驱动程序级别直接丢弃数据包，而不会浪费任何进一步的资源。这对于一般实施 DDoS 缓解机制或防火墙的 BPF 程序特别有用。
- `XDP_PASS` – 允许数据包向上传递到内核的网络堆栈。意思是，当前正在处理这个数据包的 CPU 现在分配一个skb，填充它，并将它向前传递到 GRO 引擎中。这相当于没有XDP程序操作的默认数据包处理行为。
- `XDP_TX` – 将数据包（可能已被修改），可以从刚到达的同一 NIC 传输网络数据包，转发到接收它的同一NIC网络接口。当很少有节点正在实施时，这通常很有用，例如，集群中具有负载平衡的防火墙，因此充当`hairpinned`负载平衡器，在 XDP BPF 中重写传入数据包后将其推回交换机。
- `XDP_REDIRECT` – 绕过正常的网络堆栈并通过另一个 NIC 将数据包重定向到网络。

### XDP 常用使用场景

- DDoS 缓解、防火墙
- 转发和负载均衡
- 堆栈前过滤/处理
- 流量采样、监测


### 使用 XDP 编写并运行一个简单的程序

### 先决条件

```
# eBPF programs need newer Kernel version newer than 5.0
# Base On libxdp and libbpf

git submodule add https://github.com/xdp-project/xdp-tools/ xdp-tools
git submodule add https://github.com/libbpf/libbpf/ libbpf

# Dependencies Packages on Debian
apt-get install -y clang llvm libelf-dev libpcap-dev gcc-multilib build-essential make libbpf-dev linux-perf bpftool linux-headers-$(uname -r) net-tools tcpdump

```

一个简单的 C 语言 XDP 程序

```c
#include <linux/bpf.h>
#include <bpf/bpf_helpers.h>

SEC("xdp_drop")
int xdp_drop_prog(struct xdp_md *ctx)
{
    return XDP_DROP；
}

char _license[] SEC("license") = "GPL";
```
`linux/bpf.h`头文件由 `linux-headers` 包提供，它定义了所有受支持的 BPF 帮助程序和 `xdp_actions` 例如：上面的示例中`XDP_DROP`。

`bpf/bpf_helpers.h`头文件由 `libbpf-devel` 包提供，该提供了一些有用的 eBPF SEC宏，本示例中使用的宏SEC("xdp_drop")。SEC 是`section` 的缩写，用于将已编译对象的片段放置在不同的 `ELF` 部分。
接下来`xdp_drop_prog()`函数有一个参数`struct xdp_md *ctx`，我们还没有使用过。我稍后再说。此函数直接返回XDP_DROP，这意味着我们将丢弃所有传入的数据包。

最后一行正式指定与该程序关联的许可证。验证者将使用此信息来强制执行此类限制。一些 eBPF 程序只能由 GPL 许可的程序访问。

### 构建BPF对象

```
clang -O2 -g -Wall -target bpf -c xdp_drop.c -o xdp_drop.o

```

### 加载BPF对象

```
bpftool net detach xdpgeneric dev eth0
rm -f /sys/fs/bpf/xdp_drop
bpftool prog load xdp_drop.o /sys/fs/bpf/xdp_drop
bpftool net attach xdpgeneric pinned /sys/fs/bpf/xdp_drop dev eth0
```

### 一个简单的L4 LoadBalancer的程序

接下来我们在此基础上做更多操作，来实现一个简单的L4 LoadBalancer的程序

在此之前我们需要了解关于网络TCP/IP的知识，首先需要知道**Ethernet, IP and TCP Headers**是如何构造的

具体可以阅读这篇[【what-are-ethernet-ip-and-tcp-headers-in-wireshark-captures】](http://networkstatic.net/what-are-ethernet-ip-and-tcp-headers-in-wireshark-captures/)

L4 LoadBalancer的核心是在TCP数据在链路传输过程中，变更数据包的`Ethernet Header`, `IP Header`信息来实现。

- 通过在`Client`端发起`http requset`的时候,LB在XDP探测到`ip->saddr`为Client端时，轮询（简单实现）变更该`http Request`的`iph->daddr`目的IP为后端`BACKEND_A`、`BACKEND_B`、`BACKEND_C`其中一个。

- 后端`BACKEND_A`、`BACKEND_B`、`BACKEND_C`其中一个在回复`http reponse`给`Client`客户端的时候，同样XDP在探测到，`iph->saddr`为`BACKEND_A`、`BACKEND_B`、`BACKEND_C`其中一个时，此时将`iph->daddr`目的IP变更为`Client`客户端IP，同时`Ethernet Header`中的`Dest MAC`变更为`Client`客户端的MAC地址

- 将`http Request`的`iph->saddr`源IP变更为LB的IP，`eth->h_source`变更为LB的MAC地址。

- 最后`IP Header Checksum`重新校验。


![TCP L4 termination 负载均衡](http://arthurchiao.art/assets/img/intro-to-modern-lb/l4-termination-lb.png)

### 代码实现

`xdp_lb_kern.c`

```c
#include "xdp_lb_kern.h"

#define IP_ADDRESS(x) (unsigned int)(172 + (17 << 8) + (0 << 16) + (x << 24))

#define BACKEND_A 3
#define BACKEND_B 4
#define BACKEND_C 5
#define CLIENT 6
#define LB 2
#define MAX_COUNT 3
int k = 1;

SEC("xdp_lb")
int xdp_load_balancer(struct xdp_md *ctx)
{
    void *data = (void *)(long)ctx->data;
    void *data_end = (void *)(long)ctx->data_end;

    struct ethhdr *eth = data;
    if (data + sizeof(struct ethhdr) > data_end)
        return XDP_ABORTED;

    if (bpf_ntohs(eth->h_proto) != ETH_P_IP)
        return XDP_PASS;

    struct iphdr *iph = data + sizeof(struct ethhdr);
    if (data + sizeof(struct ethhdr) + sizeof(struct iphdr) > data_end)
        return XDP_ABORTED;

    if (iph->protocol != IPPROTO_TCP)
        return XDP_PASS;

    int flag = 1;

    if (iph->saddr == IP_ADDRESS(CLIENT))
    {
        bpf_printk("Got http requset from %x",iph->saddr);
        char be = BACKEND_A;
        if(bpf_ktime_get_ns() % 2 ) {
                be = BACKEND_B;
        }
        if(k == MAX_COUNT) {
                k = 1;
                be = BACKEND_C;
        }
        k++;

        iph->daddr = IP_ADDRESS(be);
        eth->h_dest[5] = be;

    }
    else if (iph->saddr == IP_ADDRESS(BACKEND_A) || iph->saddr == IP_ADDRESS(BACKEND_B) || iph->saddr == IP_ADDRESS(BACKEND_C))
    {
        bpf_printk("Got the http response from backend [%x]: forward to client %x", iph->saddr, iph->daddr);
        iph->daddr = IP_ADDRESS(CLIENT);
        eth->h_dest[5] = CLIENT;
    } else {
        flag = 0;
    }

    if (!flag) {
            return XDP_PASS;
    }

    iph->saddr = IP_ADDRESS(LB);
    eth->h_source[5] = LB;

    iph->check = iph_csum(iph);

    return XDP_TX;
}

char _license[] SEC("license") = "GPL";

```


`xdp_lb_kern.h`

```c
#include <stddef.h>
#include <linux/bpf.h>
#include <linux/in.h>
#include <linux/if_ether.h>
#include <linux/ip.h>
#include <bpf_helpers.h>
#include <bpf_endian.h>

static __always_inline __u16
csum_fold_helper(__u64 csum)
{
    int i;
#pragma unroll
    for (i = 0; i < 4; i++)
    {
        if (csum >> 16)
            csum = (csum & 0xffff) + (csum >> 16);
    }
    return ~csum;
}

static __always_inline __u16
iph_csum(struct iphdr *iph)
{
    iph->check = 0;
    unsigned long long csum = bpf_csum_diff(0, 0, (unsigned int *)iph, sizeof(struct iphdr), 0);
    return csum_fold_helper(csum);
}

```

### 验证结果
#### LB端

```
docker run --rm -it -v ~/lb-from-scratch:/lb-from-scratch --privileged -h lb --name lb --env TERM=xterm-color 314315960/debian-ebpf-lb

root@debian-qelzsmjskk-192.168.2.5 ~ # docker run --rm -it -v ~/lb-from-scratch:/lb-from-scratch --privileged -h lb --name lb --env TERM=xterm-color 314315960/debian-ebpf-lb
root@lb:/# cd lb-from-scratch/
root@lb:/lb-from-scratch# make clean
bpftool net detach xdpgeneric dev eth0
rm -f /sys/fs/bpf/xdp_lb
rm xdp_lb_kern.o
rm xdp_lb_kern.ll
root@lb:/lb-from-scratch#
root@lb:/lb-from-scratch#
root@lb:/lb-from-scratch# make
clang -S \
    -target bpf \
    -D __BPF_TRACING__ \
    -Ilibbpf/src\
    -Wall \
    -Wno-unused-value \
    -Wno-pointer-sign \
    -Wno-compare-distinct-pointer-types \
    -Werror \
    -O2 -emit-llvm -c -o xdp_lb_kern.ll xdp_lb_kern.c
llc -march=bpf -filetype=obj -o xdp_lb_kern.o xdp_lb_kern.ll
bpftool net detach xdpgeneric dev eth0
rm -f /sys/fs/bpf/xdp_lb
bpftool prog load xdp_lb_kern.o /sys/fs/bpf/xdp_lb
bpftool net attach xdpgeneric pinned /sys/fs/bpf/xdp_lb dev eth0
root@lb:/lb-from-scratch# ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.17.0.2  netmask 255.255.0.0  broadcast 172.17.255.255
        ether 02:42:ac:11:00:02  txqueuelen 0  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0


```

#### BACKEND端


```
docker run -d --rm --name backend-A -h backend-A --env TERM=xterm-color 314315960/hello:plain-text
docker run -d --rm --name backend-B -h backend-B --env TERM=xterm-color 314315960/hello:plain-text
docker run -d --rm --name backend-C -h backend-C --env TERM=xterm-color 314315960/hello:plain-text

root@debian-qelzsmjskk-192.168.2.5 ~/lb-from-scratch # docker ps -a
CONTAINER ID   IMAGE                        COMMAND                  CREATED         STATUS         PORTS     NAMES
d799f48deb03   314315960/debian-ebpf-lb     "bash"                   3 minutes ago   Up 3 minutes             lb
af88769da64c   debian                       "bash"                   6 days ago      Up 6 days                client
6e8d787d5a0b   314315960/hello:plain-text   "/docker-entrypoint.…"   6 days ago      Up 6 days      80/tcp    backend-C
3253e525258c   314315960/hello:plain-text   "/docker-entrypoint.…"   6 days ago      Up 6 days      80/tcp    backend-B
90a6f1043001   314315960/hello:plain-text   "/docker-entrypoint.…"   6 days ago      Up 6 days      80/tcp    backend-A

```

#### Client 端

```
root@debian-qelzsmjskk-192.168.2.5 ~ # docker run --rm -it -h client --name client --env TERM=xterm-color debian

root@client:/#
# curl BACKEND_A
root@client:/# curl 172.17.0.3
Server address: 172.17.0.3:80
Server name: backend-a
Date: 22/May/2023:09:34:05 +0000
URI: /
Request ID: 615d646849c6acd542788073c970cec8

# curl BACKEND_B
root@client:/# curl 172.17.0.4
Server address: 172.17.0.4:80
Server name: backend-b
Date: 22/May/2023:09:34:09 +0000
URI: /
Request ID: 183b5503b7cadda09e8190906a1d6ac8

# curl BACKEND_C
root@client:/# curl 172.17.0.5
Server address: 172.17.0.5:80
Server name: backend-c
Date: 22/May/2023:09:34:12 +0000
URI: /
Request ID: 81b5910c382f4729661bb9874f060329


# curl L4LB
root@client:/# curl 172.17.0.2
Server address: 172.17.0.4:80
Server name: backend-b
Date: 22/May/2023:09:36:13 +0000
URI: /
Request ID: 32555c0d2e197a83d94b6d71f453457a
root@client:/# curl 172.17.0.2
Server address: 172.17.0.5:80
Server name: backend-c
Date: 22/May/2023:09:36:17 +0000
URI: /
Request ID: 2b607e547f0d046cfe40ffe0589b382f
root@client:/# curl 172.17.0.2
Server address: 172.17.0.3:80
Server name: backend-a
Date: 22/May/2023:09:36:20 +0000
URI: /
Request ID: 1f0701a283b6c576374beb247a5ac460

```

#### trace_pipe日志输出

```
root@debian-qelzsmjskk-192.168.2.5 ~/lb-from-scratch # cat /sys/kernel/debug/tracing/trace_pipe

<idle>-0       [000] d.s. 524761.663419: bpf_trace_printk: Got http requset from 60011ac
<idle>-0       [001] d.s. 524762.623046: bpf_trace_printk: Got the http response from backend [30011ac]: forward to client 20011ac
<idle>-0       [001] d.s. 524762.623084: bpf_trace_printk: Got http requset from 60011ac
<idle>-0       [001] d.s. 524763.871619: bpf_trace_printk: Got the http response from backend [30011ac]: forward to client 20011ac
<idle>-0       [001] d.s. 524763.871640: bpf_trace_printk: Got http requset from 60011ac
<idle>-0       [000] d.s. 524764.990974: bpf_trace_printk: Got the http response from backend [50011ac]: forward to client 20011ac
<idle>-0       [000] d.s. 524764.990996: bpf_trace_printk: Got http requset from 60011ac
<idle>-0       [001] d.s. 524766.527335: bpf_trace_printk: Got the http response from backend [30011ac]: forward to client 20011ac
<idle>-0       [001] d.s. 524766.527412: bpf_trace_printk: Got http requset from 60011ac
<idle>-0       [001] d.s. 524767.039258: bpf_trace_printk: Got the http response from backend [40011ac]: forward to client 20011ac
<idle>-0       [001] d.s. 524767.039279: bpf_trace_printk: Got http requset from 60011ac

```


### 总结
以上就是一个基于XDP中XDP_TX的L4LB负载均衡简单实现原理，对于ebpf的XDP的学习和运用有一定的了解，更为复杂和丰富的功能实现，可以关注Cilium周边生态项目。

### 参考文档
[https://github.com/lizrice/lb-from-scratch](https://github.com/lizrice/lb-from-scratch)

[http://arthurchiao.art/blog/intro-to-modern-lb-and-proxy-zh/](http://arthurchiao.art/blog/intro-to-modern-lb-and-proxy-zh/)

[https://github.com/w180112/ebpf_example](https://github.com/w180112/ebpf_example)

[https://github.com/ENSREG/tinyLB](https://github.com/ENSREG/tinyLB)

[https://github.com/ark-7/arkLB](https://github.com/ark-7/arkLB)

[https://github.com/xdp-project/xdp-tutorial](https://github.com/xdp-project/xdp-tutorial)

[http://networkstatic.net/what-are-ethernet-ip-and-tcp-headers-in-wireshark-captures/](http://networkstatic.net/what-are-ethernet-ip-and-tcp-headers-in-wireshark-captures/)

[https://docs.cilium.io/en/stable/bpf/progtypes/#xdp](https://docs.cilium.io/en/stable/bpf/progtypes/#xdp)

[https://arthurchiao.art/blog/cilium-bpf-xdp-reference-guide-zh/](https://arthurchiao.art/blog/cilium-bpf-xdp-reference-guide-zh/)
