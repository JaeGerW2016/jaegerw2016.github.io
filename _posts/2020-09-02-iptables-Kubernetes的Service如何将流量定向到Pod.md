---
layout:     post
title:      iptables：Kubernetes的Service如何将流量定向到Pod
subtitle: 	kube-proxy组件如何使用iptables将service流量负载均衡到pod的过程
date:       2020-09-02
author:     J
catalog: true
tags:
    - kubernetes
---



为了支持Kubernetes集群的Pod的水平扩展和高可用性，Kubernetes抽象出Service的概念。Service是对一组Pod的抽象，它会根据访问策略（如负载均衡策略）来访问这组Pod。

我们这里将重点介绍Kubernetes的Service的其中ClusterIP类型。

Service在很多情况下只是一个概念，而真正将Service的作用落实的是背后Kube-proxy的服务进程，只有理解了Kube-proxy的原理和机制，我们才能真正理解Service背后实现的逻辑。

Service的ClusterIP的概念是Kube-proxy通过Linux本身的Iptables的NAT转换实现，Kuber-proxy的运行过程中动态创建与Service相关的Iptables规则。

这里我们展开一小部分的内容进行展开，目标是实现Service负载均衡的Iptables的规则，我们就一个简单的Service为例子：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: http-server
spec:
  clusterIP: 10.100.100.100
  selector:
    component: http-server
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 8080
```



Kubernetes为每个Pod创建一个Namespace，在Linux的Namespace内可以有自己独立的路由表及独立的Iptables/Netfilter设置来提供包转发、NAT及IP包过滤等功能。

我们为了简化模型，在独立的namespace里运行 go语言写的[http-server](https://github.com/JaeGerW2016/simple-http-server)来模拟Kubernetes中的"Pod"

### 前提

Linux系统具有路由转发功能，需要配置一个Linux的内核参数`net.ipv4.ip_forward`。这个参数指定了Linux系统当前对路由转发功能的支持情况；其值为0时表示禁止进行IP转发；如果是1,则说明IP转发功能已经打开。

```shell
sudo sysctl --write net.ipv4.ip_forward=1
```

### 创建虚拟设备并在网络名称空间中运行HTTP服务器

现在我们需要

- 创建两个`namespace`: `netns_011`和`netns_021`

- 创建虚拟网桥`br1` 并分配ip `10.0.0.1`
- 给两个namespace配置DNS `8.8.8.8`
- 创建2对Veth设备对,`veth_ns_011`和`veth_ns_021`绑定连接网桥`br1`
- 分配`10.0.0.11`给 `veth_011`
- 分配`10.0.0.21`给 `veth_021`
- 分别在两个`namespace`中设置默认路由

```shell
sudo ip netns add netns_011
sudo ip netns add netns_0212

sudo ip link add dev br1 type bridge
sudo ip address add 10.0.0.1/24 dev br1
sudo ip link set br1 up

echo '8.8.8.8' | sudo tee -a /etc/netns/netns_011/resolv.conf
echo '8.8.8.8' | sudo tee -a /etc/netns/netns_021/resolv.conf
#新生成的namespace默认只有回环lo设备（默认是down状态）
sudo ip netns exec netns_011 ip link set dev lo up
sudo ip netns exec netns_021 ip link set dev lo up

sudo ip link add dev veth_011 type veth peer name veth_ns_011
sudo ip link add dev veth_021 type veth peer name veth_ns_021

sudo ip link set dev veth_011 master br1
sudo ip link set dev veth_021 master br1

sudo ip link set dev veth_ns_011 netns netns_011
sudo ip link set dev veth_ns_021 netns netns_021
sudo ip nents exec netns_011 ip link set veth_ns_011 name eth0
sudo ip nents exec netns_021 ip link set veth_ns_021 name eth0

sudo ip netns exec netns_011 ip link set dev eth0 up
sudo ip netns exec netns_021 ip link set dev eth0 up

sudo ip netns exec netns_011 ip address add 10.0.0.11/24 dev eth0
sudo ip netns exec netns_021 ip address add 10.0.0.21/24 dev eth0

sudo ip netns exec netns_011 ip route add default via 10.0.0.1
sudo ip netns exec netns_021 ip route add default via 10.0.0.1
```

创建Iptables规则以允许流量进出`br1`设备

> 如果本身的Iptables规则比较严格，需要新增以下两条规则

```shell
sudo iptables --table filter --append FORWARD --in-interface br1 --jump ACCEPT
sudo iptables --table filter --append FORWARD --out-interface br1 --jump ACCEPT
```

创建另一条Iptables的nat表的POSTROUTING规则来伪装源地址来自10.0.0.0/24的请求：

```shell
sudo iptables --table nat --append POSTROUTING --source 10.0.0.0/24 --jump MASQUERADE
```

接下来就是分别在`netns_011`和`netns_021`的namespace下启动HTTP Server

```shell
sudo ip netns exec netns_011 ./http-server
sudo ip netns exec netns_021 ./http-server
```

环境如下图：

![mmexport1599039106845.jpg](https://i.loli.net/2020/09/02/Wa6UubzMmd8SfXv.jpg)

检查HTTP Server服务是否可以 正常访问

```shell
#host network namespace
curl 10.0.0.11:8080
curl 10.0.0.21：8080

# netns_011 network namespace
sudo ip netns exec netns_011 curl 10.0.0.21:8080

# netns_021 network namespace
sudo ip netns exec netns_021 curl 10.0.0.11:8080
```

结果：

```shell
⚡ root@raspberrypi  ~  curl 10.0.0.11:8080
2020-09-02 10:47:01, you're running http-server on 10.0.0.11
⚡ root@raspberrypi  ~  curl 10.0.0.21:8080
2020-09-02 10:47:09, you're running http-server on 10.0.0.21
⚡ root@raspberrypi  ~  ip netns exec netns_011 curl 10.0.0.21:8080
2020-09-02 10:47:46, you're running http-server on 10.0.0.21
⚡ root@raspberrypi  ~  ip netns exec netns_021 curl 10.0.0.11:8080
2020-09-02 10:47:54, you're running http-server on 10.0.0.11
```

### 在Iptables中添加虚拟IP

当Kubernetes在创建Service时会同时默认创建ClusterIP给Service。从概念上讲，ClusterIP就是虚拟IP，在kube-proxy启用Iptables模式时，kube-proxy负责创建Iptables规则来处理来自Pod、外部请求。

通过以下命令在iptables的nat表新增一条新链`HTTP-SERVICE`

```SHELL
sudo iptables --table nat --new HTTP-SERVICE
```

接下来，我们需要在`PREROUTING`和`OUTPUT`链中添加规则跳转`HTTP-SERVICE`链：

```shell
sudo iptables --table nat --append PREROUTING --jump HTTP-SERVICE
sudo iptables --table nat --append OUTPUT --jump HTTP-SERVICE
```

此时，我们可以在`HTTP-SERVICE`链中创建一个规则来处理虚拟IP，我们的虚拟IP(模拟ClusterIP)，`10.100.100.100`，同时创建一个规则：

```shell
sudo iptables \
	--tables nat \
	--append HTTP-SERVICE \
	--destination 10.100.100.100 \
	--protocol tcp \
	--match tcp \
	--dport 8080 \
	--jump DNAT \
	--to-destination 10.0.0.11:8080
```

通过执行以下请求 虚拟IP:

```shell
⚡ root@raspberrypi  ~  curl 10.100.100.100:8080
2020-09-03 03:38:48, you're running http-server on 10.0.0.11
⚡ root@raspberrypi  ~  ip netns exec netns_011 curl 10.100.100.100:8080
^C
✘ ⚡ root@raspberrypi  ~  ip netns exec netns_021 curl 10.100.100.100:8080
2020-09-03 04:01:06, you're running http-server on 10.0.0.11

```

可以看出来在`netns_011`的namespace访问虚拟IP失败，这个是由于我们的请求离开`netns_011`,其源地址`10.0.0.11`，请求将发送给`10.100.100.100`，这个期间iptables规则会进行DNAT转换`10.100.100.100`至`10.0.0.11`, 这个请求最终被定向到请求来源本身。

> 如果需要namespace的网络设备通过虚拟IP来访问自身，需要在桥接设备（网桥）上针对每个端口启用`hairpin`模式，有一种方式：可以在网桥上启用混杂模式，它将所有连接端口视为已启用`hairpin`模式

``` shell
# br1 桥接设备启用混杂模式
⚡ root@raspberrypi  ~  ip link set br1 promisc on 
⚡ root@raspberrypi  ~  ip netns exec netns_011 curl 10.100.100.100:8080
2020-09-03 04:14:45, you're running http-server on 10.0.0.11
```

### 将iptables规则与kube-proxy对齐

Kube-proxy会给每个Service使用哈希创建“KUBE-SVC-<hash>”链，并在nat表中将`KUBE-SERVICE`链中每个目标地址是service的数据包导入这个'KUBE-SVC-<hash>'链，如果此时endpoint尚未创建，‘KUBE-SVC-<hash>’链中没有规则，任何incoming packets在规则匹配失败后被`KUBE-MARK-DROP`,在iptables的filter中会被`KUBR-FIRWAL`丢弃。

`KUBE_FIREWALL` 内容如下，就是直接丢弃所有报文：

```shell
Chain KUBE-FIREWALL (2 references)
target     prot opt source               destination         
DROP       all  --  anywhere             anywhere             /* kubernetes firewall for dropping marked packets */ mark match 0x8000/0x8000
```

ClusterIP类型的service链转发流程：
```shell
KUBE-SERVICE ----- KUBE-SVC-<hash> ----- KUBE-SEP-<hash>
```

```shell
sudo iptables --table nat --append PREROUTING --jump KUBE-SERVICE
sudo iptables --table nat --append OUTPUT --jump KUBE-SERVICE

sudo iptables \
	--table nat \
	--new KUBE-SERVICE

sudo iptables \
	--table nat \
	--append KUBE-SERVICE \
	--destination 10.100.100.100 \
	--protocol tcp \
	--match comment \
	--comment "default/http-server: cluster IP" \
	--match tcp \
	--dport 8080 \
	--jump KUBE-SVC-HVYO5BEGEF5HC7MD7
```

```SHELL
sudo iptables \
	--table nat \
	--new KUBE-SVC-HVYO5BEGEF5HC7MD7
	
sudo iptables \
	--table nat \
	--append KUBE-SVC-HVYO5BEGEF5HC7MD7 \
	--match comment \
	--comment "default/http-server:" \
	--jump KUBE-SEP-ESZGVIJET5GN2KKU
```

```shell
sudo iptables \
	--table nat \
	--new KUBE-SEP-ESZGVIJET5GN2KKU

sudo iptables \
	--table nat \
	--append KUBE-SEP-ESZGVIJET5GN2KKU \
	--protocol tcp \
	--match comment \
	--comment "default/http-server:" \
	--jump DNAT \
	--to-destination 10.0.0.11:8080
```

我们再尝试请求虚拟IP `10.100.100.100`

```shell
⚡ root@raspberrypi  ~  curl 10.100.100.100:8080                                    
2020-09-03 06:21:27, you're running http-server on 10.0.0.11
⚡ root@raspberrypi  ~  ip netns exec netns_021 curl 10.100.100.100:8080
2020-09-03 06:24:11, you're running http-server on 10.0.0.11
```

### 使用iptables为虚拟IP提供负载均衡

基于以上的iptables规则，我们需要针对`netns_021`的http-server添加新的链和规则：

```shell
sudo iptables \
	--table nat \
	--new KUBE-SEP-KJLGOIJET5GN2EKJ

sudo iptables \
	--table nat \
	--append KUBE-SEP-KJLGOIJET5GN2EKJ \
	--protocol tcp \
	--match comment \
	--comment "default/http-server:" \
	--jump DNAT \
	--to-destination 10.0.0.21:8080
```

然后我们还需要在`KUBE-SVC-HVYO5BEGEF5HC7MD7`链中添加另一条规则，以随机跳转到`KUBE-SEP-KJLGOIJET5GN2EKJ`

```shell
sudo iptables \
	--table nat \
	--insert KUBE-SVC-HVYO5BEGEF5HC7MD7 1 \
	--match comment \
	--comment "default/http-server:" \
	--match statistic \
	--mode random \
	--probability 0.5 \
	--jump KUBE-SEP-KJLGOIJET5GN2EKJ
```

> 需要注意的是 这条规则是insert插入到`KUBE-SVC-HVYO5BEGEF5HC7MD7`的第一条的位置。

我们知道iptables的匹配是顺序匹配，匹配到之后就不会执行后面的规则，因此，首先有了这条50%的机率会匹配规则，如果成功，iptables会跳至`KUBE-SEP-KJLGOIJET5GN2EKJ`,如果失败，则匹配下一条规则，跳至`KUBE-SEP-ESZGVIJET5GN2KKU`。

常见的误区就是给这2条KUBE-SEP规则都加上`--mode random --probability 0.5`的配置，这样会出现一种情况：

- iptables 匹配第一条规则（`--jump KUBE-SEP-KJLGOIJET5GN2EKJ`），有50%概率失败
- iptables 匹配第二条规则（`--jump KUBE-SEP-ESZGVIJET5GN2KKU`），有50%概率失败

基于以上这种情况：虚拟IP将不会DNAT转发到任何一个Pod，这个就不是我们想要的负载均衡的效果。



### 验证：

我们运行一些命令，将看到iptables将我们对虚拟IP`10.100.100.100`的请求随机转发到`10.0.0.11`和`10.0.0.21`

```shell
⚡ root@raspberrypi  ~  curl 10.100.100.100:8080
2020-09-03 06:37:41, you're running http-server on 10.0.0.21
⚡ root@raspberrypi  ~  curl 10.100.100.100:8080
2020-09-03 06:37:42, you're running http-server on 10.0.0.11
⚡ root@raspberrypi  ~  curl 10.100.100.100:8080
2020-09-03 06:37:44, you're running http-server on 10.0.0.21
⚡ root@raspberrypi  ~  curl 10.100.100.100:8080
2020-09-03 06:37:45, you're running http-server on 10.0.0.21
⚡ root@raspberrypi  ~  curl 10.100.100.100:8080
2020-09-03 06:37:46, you're running http-server on 10.0.0.11
```

以上就是我对iptables在Kube-proxy组件上对基于ClusterIP类型的Service是如果进行数据包转发及负载均衡的简单的探索与学习。
