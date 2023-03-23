---
title: Linux 网络虚拟化：Network Namespace 初探
date: 2019-07-24 17:23
tag: docker
---

在本文中我们将尝试理解Linux Network Namespace及相关Linux内核网络设备的概念，以及Docker容器网络模型的部分实现

我们在使用Docker容器时，了解过Docker是用linux network namespace实现的容器网络隔离的。使用docker时，在物理主机或虚拟机上会有一个docker0的linux bridge，brctl show时能看到 docker0上绑定veth网络设备：

```shell
[root@localhost ~]# ip link show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
5: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default 
    link/ether 02:42:fa:3f:86:7e brd ff:ff:ff:ff:ff:ff
138: veth5843bf2@if137: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP mode DEFAULT group default 
    link/ether f2:31:9d:e6:b7:16 brd ff:ff:ff:ff:ff:ff link-netnsid 2
[root@localhost ~]# brctl show
bridge name	bridge id		STP enabled	interfaces
docker0		8000.0242fa3f867e	no		veth5843bf2
```

linux network namespace是实现网络虚拟化的重要功能，它能创建多个相互隔离的网络空间，每个网络名字空间都有独立的网络配置，比如：网络设备、路由表等。

针对网络配置相关的操作需要借助于`ip` 命名，而这个命令来自于`iproute2`包，系统一般默认安装

`ip` 命令管理的功能很多， 和 network namespace 有关的操作都是在子命令 `ip netns` 下进行的，可以通过 `ip netns help` 查看所有操作的帮助信息。

```shell
[root@localhost ~]# ip netns help
Usage: ip netns list
       ip netns add NAME
       ip netns set NAME NETNSID
       ip [-all] netns delete [NAME]
       ip netns identify [PID]
       ip netns pids NAME
       ip [-all] netns exec [NAME] cmd ...
       ip netns monitor
       ip netns list-id
```

####  创建2个单独的网络命名空间

```shell
[root@localhost ~]# ip netns add net01
[root@localhost ~]# ip netns add net02
[root@localhost ~]# 
[root@localhost ~]# 
[root@localhost ~]# ip netns list
net02
net01
```

#### 查看网络设备

```shell
[root@localhost ~]# ip netns exec net01 ip addr
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
[root@localhost ~]# ip netns exec net02 ip addr
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
[root@localhost ~]# ip netns exec net01 ip link set lo up
[root@localhost ~]# ip netns exec net02 ip link set lo up
[root@localhost ~]# ip netns exec net01 ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
[root@localhost ~]# 
[root@localhost ~]# ip netns exec net02 ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever

```

#### network namespace 之间通信

linux 提供了 `veth pair` 。可以把 `veth pair` 当做是双向的 pipe（管道），从一个方向发送的网络数据，可以直接被另外一端接收到；或者也可以想象成两个 namespace 直接通过一个特殊的虚拟网卡连接起来，可以直接通信。

创建`veth pair`

```shell
[root@localhost ~]# ip link add veth01 type veth peer name veth02
[root@localhost ~]# ip a 
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
5: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 02:42:fa:3f:86:7e brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:faff:fe3f:867e/64 scope link 
       valid_lft forever preferred_lft forever
143: veth02@veth01: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether ee:71:28:fa:dc:ad brd ff:ff:ff:ff:ff:ff
144: veth01@veth02: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether fa:18:55:74:50:32 brd ff:ff:ff:ff:ff:ff
```

`veth pair` 分别放到已经两个 `namespace`

```
[root@localhost ~]# ip link set veth01 netns net01
[root@localhost ~]# ip link set veth02 netns net02
[root@localhost ~]# 
[root@localhost ~]# ip a 
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
    link/ether 00:50:56:82:08:e1 brd ff:ff:ff:ff:ff:ff
5: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 02:42:fa:3f:86:7e brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:faff:fe3f:867e/64 scope link 
       valid_lft forever preferred_lft forever
```

给这对 `veth pair` 配置上 ip 地址，并启用它们

```
[root@localhost ~]# ip netns exec net01 ip link set veth01 up
[root@localhost ~]# ip netns exec net01 ip addr add 10.0.0.1/24 dev veth01
[root@localhost ~]# 
[root@localhost ~]# ip netns exec net02 ip link set veth02 up
[root@localhost ~]# ip netns exec net02 ip addr add 10.0.0.2/24 dev veth02
```

#### 验证2个namespace的连通性

```
[root@localhost ~]# ip netns exec net01 ping -c 3 10.0.0.2 
PING 10.0.0.2 (10.0.0.2) 56(84) bytes of data.
64 bytes from 10.0.0.2: icmp_seq=1 ttl=64 time=0.142 ms
64 bytes from 10.0.0.2: icmp_seq=2 ttl=64 time=0.098 ms
64 bytes from 10.0.0.2: icmp_seq=3 ttl=64 time=0.087 ms

--- 10.0.0.2 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2000ms
rtt min/avg/max/mdev = 0.087/0.109/0.142/0.023 ms
[root@localhost ~]# ip netns exec net02 ping -c 3 10.0.0.1
PING 10.0.0.1 (10.0.0.1) 56(84) bytes of data.
64 bytes from 10.0.0.1: icmp_seq=1 ttl=64 time=0.078 ms
64 bytes from 10.0.0.1: icmp_seq=2 ttl=64 time=0.091 ms
64 bytes from 10.0.0.1: icmp_seq=3 ttl=64 time=0.095 ms

--- 10.0.0.1 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2000ms
rtt min/avg/max/mdev = 0.078/0.088/0.095/0.007 ms
```

#### 网络拓扑

![](https://i.loli.net/2019/07/24/5d37d7336dddb13235.png)

#### 使用 Linux bridge 连通多个namespace

多个网络设备通信，我们首先想到的交换机和路由器。Linux Bridge 同时可以提供二层虚拟交换机和三层路由网络功能，我们还是用 `ip` 命令来完成所有的操作。

#### 二层网络

```shell
root@ubuntu:~# ip netns add ns1
root@ubuntu:~# ip netns add ns2
root@ubuntu:~# ip link add veth1 type veth peer name eth1
root@ubuntu:~# ip link set eth1 netns ns1
root@ubuntu:~# ip link add veth2 type veth peer name eth1
root@ubuntu:~# ip link set eth1 netns ns2
root@ubuntu:~# ip netns exec ns1 ip addr add 172.18.1.10/24 dev eth1
root@ubuntu:~# ip netns exec ns2 ip addr add 172.18.1.20/24 dev eth1 
root@ubuntu:~# ip netns exec ns1 ip link set dev eth1 up
root@ubuntu:~# ip netns exec ns2 ip link set dev eth1 up
root@ubuntu:~# ip link add br0 type bridge
root@ubuntu:~# ip link set dev veth1 master br0
root@ubuntu:~# ip link set dev veth2 master br0
root@ubuntu:~# ip link set dev br0 up
root@ubuntu:~# ip link set dev veth1 up
root@ubuntu:~# ip link set dev veth2 up
root@ubuntu:~# ip netns exec ns1 ping -c 3 172.18.1.20
PING 172.18.1.20 (172.18.1.20) 56(84) bytes of data.
64 bytes from 172.18.1.20: icmp_seq=1 ttl=64 time=0.038 ms
64 bytes from 172.18.1.20: icmp_seq=2 ttl=64 time=0.107 ms
64 bytes from 172.18.1.20: icmp_seq=3 ttl=64 time=0.067 ms

--- 172.18.1.20 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2001ms
rtt min/avg/max/mdev = 0.038/0.070/0.107/0.029 ms
root@ubuntu:~# ip netns exec ns2 ping -c 3 172.18.1.10
PING 172.18.1.10 (172.18.1.10) 56(84) bytes of data.
64 bytes from 172.18.1.10: icmp_seq=1 ttl=64 time=0.107 ms
64 bytes from 172.18.1.10: icmp_seq=2 ttl=64 time=0.030 ms
64 bytes from 172.18.1.10: icmp_seq=3 ttl=64 time=0.030 ms

--- 172.18.1.10 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 1998ms
rtt min/avg/max/mdev = 0.030/0.055/0.107/0.037 ms
```

#### 三层网络

在主机上能ping通容器的话需要给`br0`加ip,容器中ping主机，需要在容器里加默认路由指向下一跳指向`br0`地址

```shell
root@ubuntu:~# ip addr add 172.18.1.1/24 dev br0
root@ubuntu:~# 
root@ubuntu:~# 
root@ubuntu:~# route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         192.168.2.1     0.0.0.0         UG    0      0        0 ens33
172.18.1.0      0.0.0.0         255.255.255.0   U     0      0        0 br0
```

```shell
root@ubuntu:~# ping -c 3 172.18.1.10
PING 172.18.1.10 (172.18.1.10) 56(84) bytes of data.
64 bytes from 172.18.1.10: icmp_seq=1 ttl=64 time=0.017 ms
64 bytes from 172.18.1.10: icmp_seq=2 ttl=64 time=0.041 ms
64 bytes from 172.18.1.10: icmp_seq=3 ttl=64 time=0.025 ms

--- 172.18.1.10 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 1999ms
rtt min/avg/max/mdev = 0.017/0.027/0.041/0.011 ms
root@ubuntu:~# ping -c 3 172.18.1.20
PING 172.18.1.20 (172.18.1.20) 56(84) bytes of data.
64 bytes from 172.18.1.20: icmp_seq=1 ttl=64 time=0.058 ms
64 bytes from 172.18.1.20: icmp_seq=2 ttl=64 time=0.028 ms
64 bytes from 172.18.1.20: icmp_seq=3 ttl=64 time=0.062 ms

--- 172.18.1.20 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 1998ms
rtt min/avg/max/mdev = 0.028/0.049/0.062/0.016 ms

root@ubuntu:~# ip netns exec ns1 ip route add default via 172.18.1.1 dev eth1
root@ubuntu:~# 
root@ubuntu:~# ip netns exec ns2 ip route add default via 172.18.1.1 dev eth1
root@ubuntu:~#
# 192.168.2.118 是主机网卡地址
root@ubuntu:~# ip netns exec ns1 ping -c 3 192.168.2.118
PING 192.168.2.118 (192.168.2.118) 56(84) bytes of data.
64 bytes from 192.168.2.118: icmp_seq=1 ttl=64 time=0.018 ms
64 bytes from 192.168.2.118: icmp_seq=2 ttl=64 time=0.058 ms
64 bytes from 192.168.2.118: icmp_seq=3 ttl=64 time=0.055 ms

--- 192.168.2.118 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 1999ms
rtt min/avg/max/mdev = 0.018/0.043/0.058/0.019 ms
root@ubuntu:~# 
root@ubuntu:~# ip netns exec ns2 ping -c 3 192.168.2.118
PING 192.168.2.118 (192.168.2.118) 56(84) bytes of data.
64 bytes from 192.168.2.118: icmp_seq=1 ttl=64 time=0.020 ms
64 bytes from 192.168.2.118: icmp_seq=2 ttl=64 time=0.027 ms
64 bytes from 192.168.2.118: icmp_seq=3 ttl=64 time=0.026 ms

--- 192.168.2.118 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 1998ms
rtt min/avg/max/mdev = 0.020/0.024/0.027/0.005 ms
```

```shell
 ens33  root@ubuntu:~# ip netns exec ns1 ping -c 3 192.168.2.118
PING 192.168.2.118 (192.168.2.118) 56(84) bytes of data.
64 bytes from 192.168.2.118: icmp_seq=1 ttl=64 time=0.018 ms
64 bytes from 192.168.2.118: icmp_seq=2 ttl=64 time=0.058 ms
64 bytes from 192.168.2.118: icmp_seq=3 ttl=64 time=0.055 ms

--- 192.168.2.118 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 1999ms
rtt min/avg/max/mdev = 0.018/0.043/0.058/0.019 ms
root@ubuntu:~# 
root@ubuntu:~# ip netns exec ns2 ping -c 3 192.168.2.118
PING 192.168.2.118 (192.168.2.118) 56(84) bytes of data.
64 bytes from 192.168.2.118: icmp_seq=1 ttl=64 time=0.020 ms
64 bytes from 192.168.2.118: icmp_seq=2 ttl=64 time=0.027 ms
64 bytes from 192.168.2.118: icmp_seq=3 ttl=64 time=0.026 ms

--- 192.168.2.118 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 1998ms
```
```shell
ens33 192.168.2.118/24
br0  172.18.1.1/24
 |
 |          +-------------+   
 |-veth1 <--|--> eth1 ns1 | 172.18.1.10/24    
 |          |-------------+   
 |-veth2 <--|--> eth1 ns2 | 172.18.1.20/24   
 |          +-------------+    
```

以上操作就是docker bridge模式的网络模型

当然要实现每个 namespace 对外网的访问还需要额外的配置（设置默认网关，开启 ip_forward，为网络添加 NAT 规则等）。