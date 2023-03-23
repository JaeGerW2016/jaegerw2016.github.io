---
layout:     post
title:     对TCP连接中TIME_WAIT的理解及对高并发的调优实践
subtitle: 	保证ingress其充分发挥高性能的优势
date:       2022-02-10
author:     J
catalog: true
tags:
    - Prometheus
    - Grafana
---
最近在给 Gateway API网关做压测，模拟高并发的场景下，TCP连接状态中会出现批量的TIME_WAIT状态的连接

![1644397008067](https://www.imgsm.com/images/2022/02/10/1644397008067.png)

持续观察一段时间之后，TIME_WAIT的连接部分被回收，Gateway API网关的负载、端口均正常，服务未出现5XX或者新建立 TCP 连接会出错，address already in use : connect 异常

> *统计主机上的各种连接状态的TCP连接数量*
>
> ``` shell
> $ netstat -ant | awk '/^tcp/ {++S[$NF]} END {for(a in S) print a, S[a]}'
> LISTEN 26
> ESTABLISHED 227
> TIME_WAIT 87
> ```

**问题分析**

TIME_WAIT 状态 TCP 连接存在，其本质原因是什么?

首先，为什么要有time wait状态 这里就涉及TCP建立连接的三次握手和断开连接四次挥手的机制相关：

- 为了可以正确处理被动关闭端可能重传的Fin报文

- 为了避免当前连接的重传报文污染重用五元组的新连接

![img](https://s4.51cto.com/oss/202008/06/f1f1edd4576c5d71b1422650920b40df.jpg)

以上这个时序图中，被动关闭端（Server 服务器）的FIN=d包可能会重传，因为主动关闭端（Client客户端）对FIN=d的回包ACK=d+1 可能丢失，这个时候可以看到主动关闭端（Client客户端）在收到FIN=d的报文并发送ACK=d+1的报文之后，进入TIME_WAIT状态，

1. 此时主动关闭端（Client客户端）并不知道ACK=d+1的是否已经到达被动关闭端（Server 服务器）

2. 被动关闭端（Server 服务器）收不到ACK=d+1的报文，不知道是FIN=d报文在传输过程中丢失还是ACK=d+1的报文在传输过程中丢失，所以无论以上2种情况中的哪一种，都需要重传FIN=d的报文，等待2MSL时间主要目的是怕最后一个 ACK=d+1报文对方没收到，那么对方在超时后将重发第三次握手的FIN=d的报文，主动关闭端接到重发的FIN=d的报文后，可以再发一个ACK=d+1应答报文。已达到单边收敛

   >附录 B：MSL 时间
   >
   >MSL，Maximum Segment Lifetime，“报文最大生存时间”
   >
   >- 任何报文在网络上存在的最长时间，超过这个时间报文将被丢弃。(IP 报文)
   >- TCP报文 (segment)是ip数据报(datagram)的数据部分。
   >
   >Tips：
   >
   >RFC 793中规定MSL为2分钟，实际应用中常用的是30秒，1分钟和2分钟等。
   >
   >2MSL，TCP 的 TIME_WAIT 状态，也称为2MSL等待状态：

time wait维持2MSL分钟级别的原因，就是为了避免进入分钟级别环路的Fin报文抵达新建连接从而造成误杀。time wait需要保证当重用五元组的新连接建立时，旧连接的所有报文均已从网络消失，包括进入环路的报文。

综上得知，TCP的time wait状态是必要的，且需要维持“那么久”的时间。这就是关于TCP time wait状态的本质。

基于以上对于高并发的场景，我们知道Nginx Ingress Controller 基于 Nginx 实现 Kubernetes Ingress API网关，在实际生产环境运行时，需要对参数进行调优，以保证其充分发挥高性能的优势。

## 内核参数调优

### 调高连接队列的大小

在高并发环境下，如果连接队列过小，则可能导致队列溢出，使部分连接无法建立。进程监听 socket 的连接队列大小受限于内核参数 `net.core.somaxconn`，调整 somaxconn 内核参数的值即可增加 Nginx Ingress 连接队列。

```shell
sysctl -w net.core.somaxconn=65535
```

### 扩大源端口范围

高并发环境将导致 Nginx Ingress 使用大量源端口与 upstream 建立连接，源端口范围从 `net.ipv4.ip_local_port_range` 内核参数中定义的区间随机选取。在高并发环境下，端口范围小容易导致源端口耗尽，使得部分连接异常。
TKE 环境创建的 Pod 源端口范围默认为32768 - 60999，建议执行以下命令扩大源端口范围，调整为1024 - 65535：

```shell
sysctl -w net.ipv4.ip_local_port_range="1024 65535"
```

### TIME_WAIT 复用

如果短连接并发量较高，所在 netns 中 TIME_WAIT 状态的连接将同样较多，而 TIME_WAIT 连接默认要等 2MSL 时长才释放，将长时间占用源端口，当这种状态连接数量累积到超过一定量之后可能会导致无法新建连接。

建议执行以下命令，为 Nginx Ingress 开启 TIME_WAIT 复用，即允许将 TIME_WAIT 连接重新用于新的 TCP 连接：

```shell
sysctl -w net.ipv4.tcp_tw_reuse=1
```

### 调大最大文件句柄数

Nginx 作为反向代理，每个请求将与 client 和 upstream server 分别建立一个连接，即占据两个文件句柄，因此理论上 Nginx 能同时处理的连接数最多是系统最大文件句柄数限制的一半。

系统最大文件句柄数由 `fs.file-max` 内核参数控制，TKE 默认值为838860。建议执行以下命令，将最大文件句柄数设置为1048576：

```shell
sysctl -w fs.file-max=1048576
```

### 配置示例

给 Nginx Ingress Controller 的 Pod 添加 initContainers 并设置内核参数。可参考以下代码示例：

```yaml
initContainers:
- name: setsysctl
image: busybox
securityContext:
privileged: true
command:
- sh
- -c
- |
sysctl -w net.core.somaxconn=65535
sysctl -w net.ipv4.ip_local_port_range="1024 65535"
sysctl -w net.ipv4.tcp_tw_reuse=1
sysctl -w fs.file-max=1048576
```

## 全局配置调优

### 调高 keepalive 连接最大请求数

Nginx 针对 client 和 upstream 的 keepalive 连接，具备 keepalive_requests 参数来控制单个 keepalive 连接的最大请求数，默认值均为100。当一个 keepalive 连接中请求次数超过默认值时，将断开并重新建立连接。

如果是内网 Ingress，单个 client 的 QPS 可能较大，例如达到10000QPS，Nginx 将可能频繁断开跟 client 建立的 keepalive 连接，并产生大量 TIME_WAIT 状态连接。为避免产生大量的 TIME_WAIT 连接，建议您在高并发环境中增大 Nginx 与 client 的 keepalive 连接的最大请求数量，在 Nginx Ingress 的配置对应 `keep-alive-requests`，可以设置为10000，详情请参见 [keep-alive-requests](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/configmap/#keep-alive-requests)。

同样，Nginx 针对 upstream 的 keepalive 连接的请求数量的配置是 `upstream-keepalive-requests`，配置方法请参见 [upstream-keepalive-requests](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/configmap/#upstream-keepalive-requests)。

### 调高 keepalive 最大空闲连接数

Nginx 针对 upstream 可配置参数 keepalive。该参数为最大空闲连接数，默认值为320。在高并发环境下将产生大量请求和连接，而实际生产环境中请求并不是完全均匀，有些建立的连接可能会短暂空闲，在空闲连接数多了之后关闭空闲连接，将可能导致 Nginx 与 upstream 频繁断连和建连，引发 TIME_WAIT 飙升。
在高并发环境下，建议将 keepalive 值配置为1000，详情请参见 [upstream-keepalive-connections](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/configmap/#upstream-keepalive-connections)。

### 调高单个 worker 最大连接数

`max-worker-connections` 控制每个 worker 进程可以打开的最大连接数，TKE 环境默认为16384。在高并发环境下建议调高该参数值，例如配置为65536，调高该值可以让 Nginx 拥有处理更多连接的能力，详情请参见 [max-worker-connections](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/configmap/#max-worker-connections)。

### 配置示例

Nginx 全局配置通过 configmap 配置（Nginx Ingress Controller 会读取并自动加载该配置）。可参考以下代码示例：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
 name: nginx-ingress-controller
# nginx ingress 性能优化: https://www.nginx.com/blog/tuning-nginx/
data:
 # nginx 与 client 保持的一个长连接能处理的请求数量，默认100，高并发场景建议调高。
 # 参考: https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/configmap/#keep-alive-requests
 keep-alive-requests: "10000"
 # nginx 与 upstream 保持长连接的最大空闲连接数 (不是最大连接数)，默认 320，在高并发下场景下调大，避免频繁建联导致 TIME_WAIT 飙升。
 # 参考: https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/configmap/#upstream-keepalive-connections
 upstream-keepalive-connections: "200"
 # 每个 worker 进程可以打开的最大连接数，默认 16384。
 # 参考: https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/configmap/#max-worker-connections
 max-worker-connections: "65536"
```

