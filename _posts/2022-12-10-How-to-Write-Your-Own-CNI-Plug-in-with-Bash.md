---
layout:     post
title:     Kubernetes 网络模型——如何使用Bash编写CNI网络插件
subtitle:  Kubernetes Networking Model and How write your own CNI plugin with Bash
date:       2022-12-10
author:     J
catalog: true
tags:
    - Kubernetes
    - CNI
---

## Kubernetes 网络模型

Kubernetes 网络模型设计背后的主要思想是，能够直接将程序从虚拟机 (VM) 转移到 Kubernetes 容器（Pod），而无需对您的应用程序进行任何更改。这提出了三个基本要求：
- 所有的Pod都可以在没有 NAT 的情况下直接相互通信。
- 所有Node都可以在没有 NAT 的情况下与所有Pod通信（反之亦然）。
- Pod将自己视为的 IP 与其他Pod将其视为的 IP 相同，扁平化网络层级结构。

这篇文章主要介绍 Kubernetes 网络模型，并介绍如何使用Bash来编写CNI网络插件。

上面的定义中提到了几个相关的组件：

*Pod*：Kubernetes 中的 pod 有点类似虚拟机(VM)有唯一的 IP 地址，运行同一个节点上的 pod 共享Node网络资源和临时存储。

*Container*：pod 是一组容器的集合，这些容器共享同一个网络命名空间。pod 内的容器通过`Pause容器`连接，这样可以使容器之间可以使用 localhost 进行通信；容器有自己独立的文件系统、CPU资源、内存资源和进程空间。需要通过创建 Pod 来创建容器。

*Node*：pod 运行在节点上，集群中包含一个或多个节点。每个 pod 的网络命名空间都会通过`veth-pair`虚拟设备接口和`cni0`网桥设备来连接节点的命名空间上，以打通网络。

在之前的文章中[Linux 网络虚拟化：Network Namespace 初探](https://jaegerw2016.github.io/2019/07/24/Linux-%E7%BD%91%E7%BB%9C%E8%99%9A%E6%8B%9F%E5%8C%96NetworkNamespace%E5%88%9D%E6%8E%A2/)介绍了`网络命名空间`，`网络命名空间之间通信`，`使用 Linux bridge 连通多个网络命名空间`

## 同一个Pod内不同container之间的通信

![](https://res.cloudinary.com/jknight/image/upload/v1624528203/BlogImages/CloudSerials/K8sNetArchP4/Pod-Inner-Communication-demo.webp)

`pause容器`，里面没有复杂的逻辑，只要不主动杀掉 Pod，`pause容器`都会一直存活，这样一来就能保证在 Pod 运行期间同一Pod 里的容器网络的稳定。
我们在同一 Pod 里所有容器里看到的网络视图，都是完全一样的，包括网络设备、IP 地址、Mac 地址等等，因为他们其实全是同一份，而这一份都来自于 Pod 第一次创建的这个`Infra container`，这样业务容器之间可以通过`localhost`来进行通讯。
由于所有的业务容器都要依赖于 pause 容器初始化的网络环境，因此在 Pod 启动时，它总是创建的第一个容器，可以说 Pod 的生命周期就是 `pause容器`的生命周期。


## 同一个Node节点内不同Pod之间的通信

![](https://res.cloudinary.com/jknight/image/upload/v1624526974/BlogImages/CloudSerials/K8sNetArchP4/Communication-between-Pods-In-A-Node.png)

Kubernetes在Node节点内网络时同样采用了docker的桥接网络结构。每个节点有一个根网络命名空间（root network namespace），负责该节点的对外通信，Linux内核提供的路由功能记录了节点eth0接口和网桥cbr0的IP地址信息。因为节点的eth0和网桥cbr0通常不在一个网段，所以这两个节点的通信就通过Linux内核提供的路由功能来实现。除了Docker里采用的是docker0网桥而Kubernetes改成了cni0网桥之外，这两者的网络结构基本相似。


## 不同Node节点之间多个Pod之间的通信

![](https://res.cloudinary.com/jknight/image/upload/v1624599374/BlogImages/CloudSerials/K8sNetArchP4/Pod-Communication-Cross-Node-Route-Mode.png)

实际的生产系统集群通常是由多个节点构成的，那么如何将这多个节点连接起来形成一个完整的相互联通的网络就成为首要问题。最简单的解决方案就是直接采用直接路由的方案，也就是在每个节点上配置集群中其他节点的地址信息，通过IP转发的方式实现跨节点Pod的通信。


了解了以上Pod不同的网络传输路径，现在需要知道的是CNI是如何实现Pod的网络配置及功能实现的。

## 集群网络构建过程及CNI插件工作内容
以`Flannel`为例、通过kubeadm的方式启动Kubernetes集群，简要总结下一个完整的Kubernetes集群网络是如何构建的：

1. 通过kubeadm初始化集群，通过Network等参数配置集群的CIDR信息
```
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

2. 当新的节点上线并尝试加入集群网络时，节点上的flanneld进程会向etcd申请本节点的Pod CIDR和其他完成通信所必须的地址信息，这些信息会被写在节点的subnet.env中。节点加入集群成功，准备好接受Master节点上scheduler的Pod调度。

3. 关于节点的规模大小，标准情况下每个节点最大仅容许同时运行110个Pod。因此每个节点的CIDR块大小约等于节点最大Pod数量的两倍，即每个节点拥有256个IP地址（“/24”）的C类IP地址网段。这么做是为了减少在节点上创建、删除Pod时IP重复使用的概率，同时避免由于IP地址耗尽带来的Pod创建失败的问题

4. 当节点接收到新的Pod创建请求时，节点上的CRI为Pod创建一个网络命名空间，之后CRI调用CNI完成网络的创建工作。

5. CNI首先会调用bridge插件检查本节点上是否存在`cni0`网桥，如果没有就创建。然后bridge插件会创建一个veth-pair，veth-pair的一端插入为Pod创建的网络命名空间中，另一端接入到cbr0网桥。换言之，节点上的`cni0`网桥是在创建节点上的第一个应用Pod时创建的。

6. CNI然后调用IPAM插件完成IP地址的申请和分配工作。IPAM会向CNI返回Pod的IP地址信息和网关IP（`cni0`网桥为默认的Pod网关）
CNI会将申请到的IP地址配置到Pod的网络命名空间中，此时Pod就拥有了集群中唯一的IP地址。如果`cni0`是当前请求中新创建的，那么bridge插件会将网关地址配置到`cni0`网桥上。

那接下就要来看下CNI是什么？已经CNI在创建Pod过程中参与做了什么？

## CNI是什么
CNI 是 CNCF 社区的一个项目，除了提供了最重要的 规范 、用来 CNI 与应用集成的lib库、实时可创建或删除已经存在的网络命名空间中CNI 插件的 CLI `cnitool`，以及 其他可引用的插件。本文发布时，最新版本为`v1.1.2`。

CNI 只关注容器的网络连接以及在容器销毁时清理/释放分配的资源，也正因为这个，即使容器发展迅速，CNI 也依然能保证简单并被 广泛支持。

根据上述的Pod创建过程中CNI的工作内容可以概括为以下内容：
- 创建Pod（容器）的 网络命名空间
- 创建veth-pair
- 配置veth-pair到Pod和Node所在网络命名空间
- 重命名 Pod所在网络命名空间veth为eth0
- 给Pod分配IP地址
- 创建cni0虚拟网桥，并配置Node端veth到cni0
- 配置cni0虚拟网桥IP并设置Pod默认路由
- 创建iptables的NAT规则
...

### CRI与CNI之间的规范

CNI 的规范涵盖了以下几部分：

- 网络配置文件格式
- 容器运行时与网络插件交互的协议
- 插件的执行流程
- 将委托其他插件的执行流程
- 返回给运行时的执行结果数据类型

### CRI与CNI交互的协议

CNI 为CRI提供 四个不同的操作：

- `ADD` - 将容器添加到网络，或修改配置
- `DEL` - 从网络中删除容器，或取消修改
- `CHECK` - 检查容器网络是否正常，如果容器的网络出现问题，则返回错误
- `VERSION` - 显示插件的版本

规范对操作的输入和输出内容进行了定义。主要几个核心的字段有：

- `CNI_COMMAND`：上面的四个操作之一
- `CNI_CONTAINERID`：容器 ID
- `CNI_NETNS`：容器的隔离域，如果用的网络命名空间，这里的值是网络命名空间的地址
- `CNI_IFNAME`：要在容器中创建的接口名，比如 eth0
- `CNI_ARGS`：执行参数时传递的参数
- `CNI_PATH`：插件可执行文件的查找路径


[![-2022-12-09-15-24-161fb3f30fedb9469a.md.png](https://youjb.com/images/2022/12/09/-2022-12-09-15-24-161fb3f30fedb9469a.md.png)](https://youjb.com/image/cW3)

[![-2022-12-09-15-25-02e83c6e3cc3fbbd65.md.png](https://youjb.com/images/2022/12/09/-2022-12-09-15-25-02e83c6e3cc3fbbd65.md.png)](https://youjb.com/image/cWG)



基于以上的内容，这里可以使用 Bash来编写一个简单CNI插件来实现基本功能：

## 使用Bash编写CNI插件

```
root@node1:/etc/cni/net.d# kubectl describe node | grep "PodCIDR"
PodCIDR:                      10.244.0.0/24
PodCIDR:                      10.244.1.0/24

#node1
# cat 10-bash-cni-plugin.conf
{
  "cniVersion": "0.3.1",
  "name": "my-cni-demo",
  "type": "bash-cni",
  "network": "10.244.0.0/16",
  "subnet": "10.244.0.0/24"
}

#node2
# cat 10-bash-cni-plugin.conf
{
  "cniVersion": "0.3.1",
  "name": "my-cni-demo",
  "type": "bash-cni",
  "network": "10.244.0.0/16",
  "subnet": "10.244.1.0/24"
}
```
配置文件`10-bash-cni-plugin.conf`中的前三个参数（`cniVersion`、`name`和`type`）是强制性的，并记录在 CNI 规范中。cniVersion用于判断插件使用的CNI版本，name只是网络名称，type指的是CNI插件可执行文件的文件名。network和参数是我们自定义的参数，在CNI规范中并没有提到，后面我们会看到网络插件subnet具体是如何使用的。
上述cni的配置文件，在不同Node上`subnet`的IP网段不同，然后需要放入`/etc/cni/net.d/`，kubelet使用此文件夹来发现`CNI`插件。

`bash-cni`接下来要做的是在主虚拟机和工作虚拟机上准备一个虚拟网桥。虚拟网桥是一种聚合来自多个网络接口的网络数据包的特殊设备。稍后，无论何时请求，我们的 CNI 插件都会将所有容器的网络接口添加到桥接器。
这使得同一主机上的容器可以自由地相互通信。网桥也可以有自己的 MAC 和 IP 地址，因此每个容器都将网桥视为插入同一网络的另一台设备。我们为主虚拟机上的网桥和工作虚拟机上的网桥保留10.244.1.1和10.244.0.1的IP地址，以下命令可用于创建和配置具有cni0名称的网桥：
```
$ sudo brctl addbr cni0
$ sudo ip link set cni0 up
$ sudo ip addr add <bridge-ip>/24 dev cni0
```
这些命令创建网桥、启用它，然后为其分配 IP 地址。最后一个命令还隐含地创建了一条路由，以便所有目标 IP 属于当前节点本地的 pod CIDR 范围的流量都将被重定向到cni0网络接口。（如前所述，所有其他软件都与网桥通信，就好像它是普通网络接口一样。）您可以通过ip route从主虚拟机和工作虚拟机运行以下命令来查看这个回指到10.244.0.0/24和10.244.1.0/24本地路由：
```
$ ip route | grep cni0
10.244.0.0/24 dev cni0  proto kernel  scope link  src 10.244.0.1

$ ip route | grep cni0
10.244.1.0/24 dev cni0  proto kernel  scope link  src 10.244.1.1
```
现在，让我们使用Bash来创建`CNI`插件本身。`CNI`插件的可执行文件必须放在/opt/cni/bin/文件夹中，其名称必须type与插件配置中的参数（bash-cni）完全一致。（将插件放入正确的文件夹后，不要忘记通过运行 使其可执行`sudo chmod +x bash-cni`。）

`bash-cni`

```shell
#!/bin/bash -e

if [[ ${DEBUG} -gt 0 ]]; then set -x; fi

exec 3>&1 # make stdout available as fd 3 for the result
exec &>> /var/log/bash-cni-plugin.log

IP_STORE=/tmp/reserved_ips # all reserved ips will be stored there

echo "CNI command: $CNI_COMMAND"

stdin=`cat /dev/stdin`
echo "stdin: $stdin"

allocate_ip(){
        r=$((RANDOM % ${#all_ips[@]}))
        for ip in "${all_ips[$r]}"
        do
                reserved=false
                for reserved_ip in "${reserved_ips[@]}"
                do
                        if [ "$ip" = "$reserved_ip" ]; then
                                reserved=true
                                break
                        fi
                done
                if [ "$reserved" = false ] ; then
                        echo "$ip" >> $IP_STORE
                        echo "$ip"
                        return
                fi
        done
}

case $CNI_COMMAND in
ADD)
        network=$(echo "$stdin" | jq -r ".network")
        subnet=$(echo "$stdin" | jq -r ".subnet")
        subnet_mask_size=$(echo $subnet | awk -F  "/" '{print $2}')
        hostnetwork=$(echo "$stdin" | jq -r ".hostnetwork")

        all_ips=$(nmap -sL $subnet | grep "Nmap scan report" | awk '{print $NF}')
        all_ips=(${all_ips[@]})
        lastIndex=$((${#all_ips[@]}-1))
        skip_first_ip=${all_ips[0]}
        skip_last_ip=${all_ips[lastIndex]}
        gw_ip=${all_ips[1]}
        reserved_ips=$(cat $IP_STORE 2> /dev/null || printf "$skip_first_ip\n$skip_last_ip\n$gw_ip\n") # reserving 10.244.0.0 and 10.244.0.255 and 10.244.0.1
        reserved_ips=(${reserved_ips[@]})
        printf '%s\n' "${reserved_ips[@]}" > $IP_STORE
        container_ip=$(allocate_ip)

        if ! ip link show cni0 &>/dev/null; then
        ip link add cni0 type bridge > /dev/null 2>&1
        ip address add "$gw_ip"/"${subnet_mask_size}" dev cni0
        ip link set cni0 up
        fi

        rand=$(date +%s%N | md5sum | head -c 6)
        host_if_name="veth$rand"
        ip link add dev ${host_if_name} type veth peer name veth1_"${rand}"
        ip link set dev ${host_if_name} up
        ip link set ${host_if_name} master cni0

        mkdir -p /var/run/netns/
        ln -sf "$CNI_NETNS" /var/run/netns/"$CNI_CONTAINERID"
        ip link set veth1_"${rand}" netns ${CNI_CONTAINERID}
        ip netns exec ${CNI_CONTAINERID} ip link set veth1_"${rand}" name ${CNI_IFNAME}
        ip netns exec ${CNI_CONTAINERID} ip link set dev lo up
        ip netns exec ${CNI_CONTAINERID} ip link set $CNI_IFNAME up
        ip netns exec ${CNI_CONTAINERID} ip addr add $container_ip/$subnet_mask_size dev $CNI_IFNAME
        ip netns exec ${CNI_CONTAINERID} ip route add default via $gw_ip dev $CNI_IFNAME

        mac=$(ip netns exec ${CNI_CONTAINERID} ip link show eth0 | awk '/ether/ {print $2}')
        echo "{
          \"cniVersion\": \"0.3.1\",
          \"interfaces\": [                                            
              {
                  \"name\": \"eth0\",
                  \"mac\": \"$mac\",                            
                  \"sandbox\": \"$CNI_NETNS\"
              }
          ],
          \"ips\": [
              {
                  \"version\": \"4\",
                  \"address\": \"$container_ip/$subnet_mask_size\",
                  \"gateway\": \"$gw_ip\",          
                  \"interface\": 0
              }
          ]
        }" >&3

;;

DEL)
        ip=$(ip netns exec $CNI_CONTAINERID ip addr show eth0 | awk '/inet / {print $2}' | sed  s%/.*%% || echo "")
        if [ ! -z "$ip" ]
        then
                sed -i "/$ip/d" $IP_STORE
        fi
        rm -f /var/run/netns/"$CNI_CONTAINERID" &>/dev/null
;;

GET)
        echo "GET not supported"
        exit 1
;;

VERSION)
echo '{
  "cniVersion": "0.3.1",
  "supportedVersions": [ "0.3.0", "0.3.1", "0.4.0" ]
}' >&3
;;

*)
  echo "Unknown cni commandn: $CNI_COMMAND"
  exit 1
;;

esac

```
脚本开头2行
```shell
exec 3>&1
exec &>> /var/log/bash-cni-plugin.log
```
第一行表示创建新的文件描述符`3`并将其重定向到`STDOUT`标准输出。

第二行表示将`STDOUT`和`STDERR`重定向输出到`/var/log/bash-cni-plugin.log`中，后续可以在该文件中查看该插件的执行日志。
后续的`echo “” >&3`需要输出内容到`STDOUT`标准输出。

CNI 插件支持四种命令`ADD`、`DEL`、`GET`和`VERSION`。CNI 插件的调用者(kubelet)必须初始化CNI_COMMAND包含所需命令的环境变量`CNI_COMMAND`,`CNI_CONTAINERID`,`CNI_NETNS`,`CNI_IFNAME`。最重要的命令是ADD，每次创建容器后执行，负责为容器分配网络接口。让我们来看看它是如何工作的。

首先首先从插件配置`10-bash-cni-plugin.conf`中检索`CNI`插件所需的值。

```shell
network=$(echo "$stdin" | jq -r ".network")       #10.244.0.0/16
subnet=$(echo "$stdin" | jq -r ".subnet")         #10.244.1.0/24
subnet_mask_size=$(echo $subnet | awk -F  "/" '{print $2}') # 24
```
我们正在获取network和subnet变量的值。我们还解析子网掩码。对于10.244.1.0/24子网，子网掩码为24。

接下来，就是需要容器（Container）分配一个container IP 地址。
```shell

    all_ips=$(nmap -sL $subnet | grep "Nmap scan report" | awk '{print $NF}')
    all_ips=(${all_ips[@]})
    lastIndex=$((${#all_ips[@]}-1))
    skip_first_ip=${all_ips[0]}
    skip_last_ip=${all_ips[lastIndex]}
    gw_ip=${all_ips[1]}
    reserved_ips=$(cat $IP_STORE 2> /dev/null || printf "$skip_first_ip\n$skip_last_ip\n$gw_ip\n") # 保留 10.244.0.0 和 10.244.0.1和10.244.0.255
    reserved_ips=(${reserved_ips[@] })
    printf '%s\n' "${reserved_ips[@]}" > $IP_STORE
    container_ip=$(allocate_ip)

```
这部分脚本内容是创建可用ip列表和保留ip列表以及`cni0`网桥ip，通过`nmap`命令扫描`subnet`子网C类地址获取256 IP地址。
其中`10.244.1.0`，`10.244.1.255`作为保留地址，`10.244.1.1`作为`cni0`网关地址。
最后，我们调用`allocate_ip函数`，它只是迭代all_ips和reserved_ips，找到第一个非保留 IP，并更新/tmp/reserved_ips文件。这是通常由 IPAM（IP 地址管理）CNI 插件完成的工作。我们分配 IP 地址的方式与主机本地 `IPAM CNI 插件`完全相同还有其他可能的分配 IP 地址的方法，例如`DHCP 服务器`分配。

接下来的两行
```shell
	mkdir -p /var/run/netns/
	ln -sf $CNI_NETNS /var/run/netns/$CNI_CONTAINERID
```
CNI 插件告诉调用者（kubelet）创建一个网络命名空间并将其传递到CNI_NETNS环境变量中。

在这里，我们正在创建一个指向网络命名空间并位于/var/run/netns/文件夹中的符号链接（这是该ip netns工具创建的）。执行这些命令后，我们将能够使用该ip netns $CNI_CONTAINERID命令访问网络命名空间。

然后我们创建一对`veth-pair`网络接口

```shell
	rand=$(date +%s%N | md5sum | head -c 6)
  host_if_name="veth$rand"
  ip link add dev ${host_if_name} type veth peer name veth1_"${rand}"
```
`veth-pair`网络接口的特性就是传输到这对设备中的一个veth设备的包会立即在另一个设备上接收。`CNI_IFNAME`由调用者（kubelet）提供并指定将分配给容器的网络接口的名称（通常为`eth0`）。第二个网络接口的名称是动态随机生成的。


第二个接口保留在Node上网络命名空间中，同时添加到网桥中，作为二层网桥设备`cni0`的一个端口。这是我们将在接下来的两行中做的事情。

```shell
	ip link set $host_if_name up
	ip link set $host_if_name master cni0
```
接下来就是配置容器网络命名空间的网络配置，

```shell
	ip link set $CNI_IFNAME netns $CNI_CONTAINERID
	ip netns exec $CNI_CONTAINERID ip link set $CNI_IFNAME up
	ip netns exec $CNI_CONTAINERID ip addr add $container_ip/$subnet_mask_size dev $CNI_IFNAME
	ip netns exec $CNI_CONTAINERID ip route add default via $gw_ip dev $CNI_IFNAME

	mac=$(ip netns exec $CNI_CONTAINERID ip link show eth0 | awk '/ether/ {print $2}')

```
我们`将$CNI_IFNAME`接口移动到容器的网络命名空间。（在这一步之后，主机命名空间中的任何人都无法直接与容器接口通信。所有通信必须仅通过主机对完成。）然后，我们将之前分配的容器 IP 分配给接口并创建默认路由将所有流量重定向到默认网关，即`cni0`网桥的 IP 地址）。MAC地址在`veth-pair`创建的时候自动生成。

以上现在所有配置都已完成。剩下的唯一事情就是将创建的网络接口的信息返回给`CNI`的调用者（`kubelet`）。这是我们在`ADD` 命令的最后一条语句`echo`所做的事情,同时用`&3文件描述符`将结果重定向到原始`STDOUT`。

其他三个`CNI`命令就简单多了，对我们来说也不是那么重要。`DEL`命令只是从`reserved_ips`列表中释放删除容器的IP地址。请注意，我们不必删除任何网络接口，因为它们会在删除容器网络命名空间后自动删除。`GET`的目的是返回一些之前创建的容器的信息，但是 `kubelet` 没有使用它，我们根本就没有具体实现它。`VERSION`打印支持的 `CNI` 版本。



### 初始化Kubernetes集群

`kubeadm-control-plane-init.sh`
```shell

#!/bin/bash

CONTROL_PLANE_NAME="control-plane"
WORKER_NODE_NAME="worker-node-2"

sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab

sudo apt update -y && sudo apt upgrade -y
sudo apt-get install -y ca-certificates apt-transport-https curl wget gnupg lsb-release apparmor apparmor-utils

sudo curl -fsSL https://download.docker.com/linux/debian/gpg | sudo apt-key add -
echo "deb [arch=amd64] https://download.docker.com/linux/debian bullseye stable" | sudo tee /etc/apt/sources.list.d/docker.list
sudo apt update -y
sudo apt install -y containerd

wget https://github.com/containerd/containerd/releases/download/v1.6.12/containerd-1.6.12-linux-amd64.tar.gz -O /tmp/containerd-1.6.12.tar.gz &>/dev/null

tar -xf /tmp/containerd-1.6.12.tar.gz && mv /tmp/bin/* /usr/bin/

sudo mkdir -p /etc/containerd
[ -e  /etc/containerd/config.toml ] && rm /etc/containerd/config.toml
sudo containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/            SystemdCgroup = false/            SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd

cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

echo 1 > /proc/sys/net/ipv4/ip_forward
lsmod | grep br_netfilter
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sudo sysctl --system


sudo apt-get update -y
sudo curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
sudo apt-get update -y
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
sudo hostnamectl set-hostname "${CONTROL_PLANE_NAME}"
#sudo hostnamectl set-hostname worker-node

kubeadm init --pod-network-cidr=10.244.0.0/16

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
export KUBECONFIG=/etc/kubernetes/admin.conf
```

`kubeadm-worker-init.sh`

```shell

#!/bin/bash

CONTROL_PLANE_NAME="control-plane"
WORKER_NODE_NAME="worker-node-2"

sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab

sudo apt update -y && sudo apt upgrade -y
sudo apt-get install -y ca-certificates apt-transport-https curl wget gnupg lsb-release apparmor apparmor-utils

sudo curl -fsSL https://download.docker.com/linux/debian/gpg | sudo apt-key add -
echo "deb [arch=amd64] https://download.docker.com/linux/debian bullseye stable" | sudo tee /etc/apt/sources.list.d/docker.list
sudo apt update -y
sudo apt install -y containerd

wget https://github.com/containerd/containerd/releases/download/v1.6.12/containerd-1.6.12-linux-amd64.tar.gz -O /tmp/containerd-1.6.12.tar.gz &>/dev/null

tar -xf /tmp/containerd-1.6.12.tar.gz && mv /tmp/bin/* /usr/bin/

sudo mkdir -p /etc/containerd
[ -e  /etc/containerd/config.toml ] && rm /etc/containerd/config.toml
sudo containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/            SystemdCgroup = false/            SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd

cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

echo 1 > /proc/sys/net/ipv4/ip_forward
lsmod | grep br_netfilter
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sudo sysctl --system


sudo apt-get update -y
sudo curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
sudo apt-get update -y
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
@sudo hostnamectl set-hostname control-plane-node
sudo hostnamectl set-hostname "${WORKER_NODE_NAME}"
sed -i '3i\127.0.0.1       "${WORKER_NODE_NAME}"\' /etc/hosts



#kubeadm join ${CONTROL_PLANE_IP}:6443 --token xxxxxxx.xxxxxxxxxxxxxx --discovery-token-ca-cert-hash sha256:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

```
### worker node join kubernetes Cluster
```shell
kubeadm token create --print-join-command  # run this on control-plane


kubeadm join ${CONTROL_PLANE_IP}:6443 --token xxxxxxx.xxxxxxxxxxxxxx --discovery-token-ca-cert-hash sha256:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```
### Node ports
#### control-plane

| Port        | Protocol | Description             |
|-------------|----------|-------------------------|
| 6443        | TCP      | API Server              |
| 2379 - 2380 | TCP      | Etcd Server Client API  |
| 10250       | TCP      | Kubelet API             |
| 10251       | TCP      | Kube Scheduler          |
| 10252       | TCP      | Kube Controller Manager |

#### Worker

| Port          | Protocol | Description       |
|---------------|----------|-------------------|
| 10250         | TCP      | Kubelet API       |
| 30000 - 32767 | TCP      | NodePort services |


### 验证初始化集群

```
root@debian-wmctmfwjmq-192.168.2.209 ~ # kubectl describe node | egrep  "^Name|^PodCIDR"
Name:               debian-vdzdtctydv
PodCIDR:                      10.244.1.0/24
PodCIDRs:                     10.244.1.0/24
Name:               debian-wmctmfwjmq
PodCIDR:                      10.244.0.0/24
PodCIDRs:                     10.244.0.0/24
```

`demo-pod.yaml`
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: busybox1
spec:
  containers:
  - name: busybox
    image: yauritux/busybox-curl
    command:
      - "/bin/ash"
      - "-c"
      - "sleep 2000"
  nodeSelector:
    kubernetes.io/hostname: debian-vdzdtctydv
---
apiVersion: v1
kind: Pod
metadata:
  name: busybox2
spec:
  containers:
  - name: busybox
    image: yauritux/busybox-curl
    command:
      - "/bin/ash"
      - "-c"
      - "sleep 2000"
  nodeSelector:
    kubernetes.io/hostname: debian-wmctmfwjmq
---
apiVersion: v1
kind: Pod
metadata:
  name: nginx1
spec:
  containers:
  - name: nginx
    image: nginx
  nodeSelector:
    kubernetes.io/hostname: debian-vdzdtctydv
---
apiVersion: v1
kind: Pod
metadata:
  name: nginx2
spec:
  containers:
  - name: nginx
    image: nginx
  nodeSelector:
    kubernetes.io/hostname: debian-wmctmfwjmq
```


```
root@debian-wmctmfwjmq-192.168.2.209 ~ # kubectl get pod -owide
NAME       READY   STATUS    RESTARTS   AGE    IP             NODE                NOMINATED NODE   READINESS GATES
busybox1   1/1     Running   0          118s   10.244.1.189   debian-vdzdtctydv   <none>           <none>
busybox2   1/1     Running   0          118s   10.244.0.116   debian-wmctmfwjmq   <none>           <none>
nginx1     1/1     Running   0          118s   10.244.1.105   debian-vdzdtctydv   <none>           <none>
nginx2     1/1     Running   0          118s   10.244.0.113   debian-wmctmfwjmq   <none>           <none>
```
### 同节点Pod之间通讯

```shell
$ sudo iptables -t filter -A FORWARD -s 10.244.0.0/16 -j ACCEPT
$ sudo iptables -t filter -A FORWARD -d 10.244.0.0/16 -j ACCEPT
```

```shell

root@debian-wmctmfwjmq-192.168.2.209 ~ # kubectl exec -it busybox1 -- ping 10.244.1.105
PING 10.244.1.105 (10.244.1.105): 56 data bytes
64 bytes from 10.244.1.105: seq=0 ttl=64 time=0.072 ms
64 bytes from 10.244.1.105: seq=1 ttl=64 time=0.071 ms
64 bytes from 10.244.1.105: seq=2 ttl=64 time=0.061 ms
64 bytes from 10.244.1.105: seq=3 ttl=64 time=0.069 ms
64 bytes from 10.244.1.105: seq=4 ttl=64 time=0.069 ms
^C
--- 10.244.1.105 ping statistics ---
5 packets transmitted, 5 packets received, 0% packet loss
round-trip min/avg/max = 0.061/0.068/0.072 ms
root@debian-wmctmfwjmq-192.168.2.209 ~ # kubectl exec -it busybox1 -- wget -qO- 10.244.1.105
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>

```

### 不同节点之间Pod通讯

```shell
$ ip route add 10.244.1.0/24 via 192.168.2.205 dev ens33 # run on control-plane
$ ip route add 10.244.0.0/24 via 192.168.2.209 dev ens33 # run on worker


iptables -t nat -N BASH_CNI_MASQUERADE &>/dev/null
iptables -t nat -A BASH_CNI_MASQUERADE -d "10.244.0.0/16" -j RETURN
iptables -t nat -A BASH_CNI_MASQUERADE -d "192.168.2.0/24" -j RETURN
iptables -t nat -A BASH_CNI_MASQUERADE -j MASQUERADE
iptables -t nat -A POSTROUTING -s "10.244.0.0/16" -j BASH_CNI_MASQUERADE

or

sudo iptables -t nat -A POSTROUTING -s 10.244.0.0/16 ! -o cni0 -j MASQUERADE
```
```shell
root@debian-wmctmfwjmq-192.168.2.209 ~ # kubectl exec -it busybox1 -- wget -qO- 10.244.0.113
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
root@debian-wmctmfwjmq-192.168.2.209 ~ # kubectl exec -it busybox2 -- wget -qO- 10.244.11.105
^Ccommand terminated with exit code 130
root@debian-wmctmfwjmq-192.168.2.209 ~ # kubectl exec -it busybox2 -- wget -qO- 10.244.1.105
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

### 同互联网公网通讯

```
root@debian-wmctmfwjmq-192.168.2.209 ~ # kubectl exec -it busybox1 -- ping 8.8.8.8
PING 8.8.8.8 (8.8.8.8): 56 data bytes
64 bytes from 8.8.8.8: seq=0 ttl=111 time=33.673 ms
64 bytes from 8.8.8.8: seq=1 ttl=111 time=33.843 ms
64 bytes from 8.8.8.8: seq=2 ttl=111 time=33.820 ms
64 bytes from 8.8.8.8: seq=3 ttl=111 time=34.047 ms
64 bytes from 8.8.8.8: seq=4 ttl=111 time=34.430 ms
^C
--- 8.8.8.8 ping statistics ---
5 packets transmitted, 5 packets received, 0% packet loss
round-trip min/avg/max = 33.673/33.962/34.430 ms
root@debian-wmctmfwjmq-192.168.2.209 ~ # kubectl exec -it busybox2 -- ping 8.8.8.8
PING 8.8.8.8 (8.8.8.8): 56 data bytes
64 bytes from 8.8.8.8: seq=0 ttl=111 time=34.053 ms
64 bytes from 8.8.8.8: seq=2 ttl=111 time=34.953 ms
64 bytes from 8.8.8.8: seq=3 ttl=111 time=34.107 ms
64 bytes from 8.8.8.8: seq=4 ttl=111 time=33.534 ms
64 bytes from 8.8.8.8: seq=5 ttl=111 time=33.965 ms
64 bytes from 8.8.8.8: seq=6 ttl=111 time=34.230 ms
^C
--- 8.8.8.8 ping statistics ---
7 packets transmitted, 6 packets received, 14% packet loss
round-trip min/avg/max = 33.534/34.140/34.953 ms
```

### 总结

至此整个CNI的工作机制和Pod的网络模型，通过这个`bash-cni`的编写有了一个系统的理解，
Kubernetes的网络始终是最复杂，相信能够对最简单的Pod网络模型有了一定的了解，现实中我们有很多CNI插件可以选择：
- `flannel`
- `calico`
- `cilium`
...

希望这篇文章能对你理解Pod网络模型及CNI的工作机制有一定帮助~~~
