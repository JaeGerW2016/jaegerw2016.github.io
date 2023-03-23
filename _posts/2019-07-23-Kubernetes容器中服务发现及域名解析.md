---
title: Kubernetes容器中服务发现及域名解析
date: 2019-07-23 09:53
tag: kubernetes
---

在 Kubernetes 中，服务发现有几种方式：
①：基于环境变量的方式
②：基于内部域名的方式

基本上，使用环境变量的方式很少，主要还是使用**内部域名**这种服务发现的方式。

其中，基于内部域名的方式，涉及到 Kubernetes 内部域名的解析，这里就需要在集群内部署CoreDNS，Kubernetes 1.13之后 作为Kubernetes默认DNS服务

### Kubernetes 中的域名是如何解析的

kubernetes上运行的容器，其域名解析和一般的Linux一样，都是根据 `/etc/resolv.conf` 文件。如下是容器中该文件的内容。

```shell
/ # cat /etc/resolv.conf 
nameserver 10.68.0.2
search default.svc.cluster.local. svc.cluster.local. cluster.local.
options ndots:5
```

nameserver 指向的kubernetes集群中coredns的svc ClusterIP 这个IP是ipvs的虚拟ip，可以直接访问，无法对其进去ping

```shell
root@k8s-master-1:~ # kubectl get svc -n kube-system
NAME                                          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                   AGE
coredns                                       ClusterIP   10.68.0.2       <none>        53/UDP,53/TCP,9153/TCP    17d
```

集群内的所有域名解析都要经过coredns的ClusterIP进行解析，不论是Kubernetes内部解析还是外部解析。

Kubernetes 中，域名的全称，必须是 `service-name.namespace.svc.cluster.local.` 这种格式，这里就需要先了解一个概念：`FQDN(Fully qualified domain name)`

FQDN是完整域名，一般来说，域名最终以`.`结束表示是FQDN，例如`google.com.`是FQDN，但`google.com`不是。

对FQDN，操作系统会直接查询DNS server。那么非FQDN呢？这里就要用到`search`和`ndots`了

### search和ndots

`ndots`表示的是域名中必须出现的`.`的个数，如果域名中的`.`的个数不小于`ndots`的值，则该域名为一个FQDN，操作系统会直接查询；如果域名中的`.`的个数小于`ndots`的值，操作系统会在`search`搜索域中进行查询。

##### 示例1：外部域名

`ndots`为5，查询的域名`google.com`不以`.`结尾，且`.`的个数少于5，依次根据搜索域查询：`google.com.default.svc.cluster.local.` -> `google.com.svc.cluster.local.` -> `google.com.cluster.local.`->`google.com.`，直到找到为止。

因此操作系统会依此在`default.svc.cluster.local. svc.cluster.local. cluster.local. lan`4个搜索域中进行了搜索，这3个搜索域是由kubernetes注入的,`lan`是node网络环境



```shell
ndots:n
	Sets a threshold for the number of dots which must appear in a name given to res_query(3) (see resolver(3)) before an initial absolute query will be made.  The default  for  n  is  1, meaning  that  if  there  are  any  dots  in a name, the name will be tried first as an absolute name before any search list elements are appended to it.  The value for this option is silently capped to 15.
```

`ndots`默认值为1，也就是说，只要域名中有一个`.`，操作系统就会认为是完整域名，直接查询DNS server。

`ndots`上限为15

##### 示例2：内部域名

Service 名称为 a 在容器内，会根据 /etc/resolve.conf 进行解析流程。选择 nameserver 10.68.0.2 进行解析，然后，用字符串 “a”，依次带入 /etc/resolve.conf 中的 search 域，进行DNS查找，分别是

```
nameserver 10.68.0.2
search default.svc.cluster.local. svc.cluster.local. cluster.local.
options ndots:5
```

依次根据搜索域查询：`a.default.svc.cluster.local.` -> `a.svc.cluster.local.` -> `a.cluster.local.` 知道找到为止。

**kubernetes之所以要设置搜索域，目的是为了方便用户访问service，同时搜索域是有优先级。**

例如，default namespace下的Pod a，如果访问同namespace下的service b，直接使用 b 就可以访问了，而这个功能，就是通过`nsSvcDomain`搜索域`default.svc.cluster.local`完成的。

类似的，对于不同namespace下的service，可以用 `${service name}.${namespace name}` 来访问，是通过`svcDomain`搜索域完成的。

`clusterDomain`设计的目的，是为了方便同域中非kubernetes上的域名访问，例如设置kubernetes的domain为`jaeger.com`，那么对于`c.jager.com`域名，直接使用`s`来访问就可以了，当然前提是当前namespace中没有一个叫做`c`的svc。

#### 为何会出现多余DNS请求的情况

上述**示例1**中,先是依次查询了下面几个域名，并且均查询了IPv4和IPv6:

- `google.com.default.svc.cluster.local.`
- `google.com.svc.cluster.local.`
- `google.com.cluster.local.`
- `google.com.`

其实真正有效nslookup的是最后一次绝对域名`google.com.` 其余的3个域名查询都是多余的DNS查询请求

**优化方式一**：具体应用配置特定的 ndots

在 Kubernetes 中，默认设置了 ndots 值为5，是因为，Kubernetes 认为，内部域名，最长为5，要保证内部域名的请求，优先走集群内部的DNS，而不是将内部域名的DNS解析请求，有打到外网的机会，Kubernetes 设置 ndots 为5是一个比较合理的行为。

如果你需要定制这个长度，最好是为自己的业务，通过`pod.Spec.DNSConfig`单独配置 `ndots`即可

```
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

**优化方式二**：Kubernetes DNS 策略

```
const (
	// DNSClusterFirstWithHostNet indicates that the pod should use cluster DNS
	// first, if it is available, then fall back on the default
	// (as determined by kubelet) DNS settings.
	DNSClusterFirstWithHostNet DNSPolicy = "ClusterFirstWithHostNet"

	// DNSClusterFirst indicates that the pod should use cluster DNS
	// first unless hostNetwork is true, if it is available, then
	// fall back on the default (as determined by kubelet) DNS settings.
	DNSClusterFirst DNSPolicy = "ClusterFirst"

	// DNSDefault indicates that the pod should use the default (as
	// determined by kubelet) DNS settings.
	DNSDefault DNSPolicy = "Default"

	// DNSNone indicates that the pod should use empty DNS settings. DNS
	// parameters such as nameservers and search paths should be defined via
	// DNSConfig.
	DNSNone DNSPolicy = "None"
)
```

这几种DNS策略，需要在Pod，或者Deployment、RC等资源中，`pod.Spec`设置 `dnsPolicy` 即可，以 Pod 为例

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

> busybox: latest镜像在进行nslookup会报错，慎用
>
> https://github.com/docker-library/busybox/issues/48

- `None`：它允许Pod忽略Kubernetes环境中的DNS设置。应使用`dnsConfig`Pod规范中的字段提供所有DNS设置 。请参阅下面的[Pod的DNS配置](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/#pod-s-dns-config)子部分
- `Default`：Pod从运行pod的节点继承名称解析配置。有关 详细信息，请参阅[相关讨论](https://kubernetes.io/docs/tasks/administer-cluster/dns-custom-nameservers/#inheriting-dns-from-the-node)

- `ClusterFirst`”：任何与配置的群集域后缀不匹配的DNS查询（例如“ `www.kubernetes.io`”）将转发到从该节点继承的上游名称服务器。群集管理员可能配置了额外的存根域和上游DNS服务器。有关 在这些情况下如何处理DNS查询的详细信息，请参阅[相关讨论](https://kubernetes.io/docs/tasks/administer-cluster/dns-custom-nameservers/#impacts-on-pods)。

- `ClusterFirstWithHostNet`：对于使用hostNetwork=true运行的Pod，您应该明确设置其DNS策略 `ClusterFirstWithHostNet`。

> **注意：** “Default”不是默认的DNS策略。如果`dnsPolicy`未明确指定，则使用“ClusterFirst”。

**优化方式三**：CoreDNS的autopath插件和DNS Cache

#### autopath

把我们多次的在 `searchdomain` 拼凑的 DNS 记录的查询在在服务器上给实现了。 避免了多次的 Client 端和 Server 端的数据交互

![](https://i.loli.net/2019/07/23/5d365b77ec86558600.png)

![](https://i.loli.net/2019/07/23/5d365b77e7f8f76574.png)

![](https://i.loli.net/2019/07/23/5d365b777f50361943.png)

![](https://i.loli.net/2019/07/23/5d365b77770ba36468.png)

下面我在一个`busybox`的容器里进行`nslookup`操作，同时在容器所在的node上`network namespace` 中抓包

```shell
root@k8s-master-1:~/k8s_manifests/busybox# kubectl exec -it nslookup-busybox sh
/ # 
/ # 
/ # nslookup google.com
Server:    10.68.0.2
Address 1: 10.68.0.2 coredns.kube-system.svc.cluster.local

Name:      google.com
Address 1: 2404:6800:4008:800::200e tsa01s07-in-x0e.1e100.net
Address 2: 216.58.200.238 tsa03s01-in-f238.1e100.net
```

```shell
root@k8s-node-3:~# docker inspect --format="{{.State.Pid}}" 7aa860011006
18033

root@k8s-node-3:~# nsenter -t 18033 -n tcpdump -i eth0 'udp and port 53'
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
01:11:09.149214 IP 172.20.3.91.42268 > 10.68.0.2.domain: 2+ PTR? 2.0.68.10.in-addr.arpa. (40)
01:11:09.155437 IP 172.20.3.91.38278 > 10.68.0.2.domain: 3+ AAAA? google.com. (28)
01:11:09.164548 IP 172.20.3.91.41062 > 10.68.0.2.domain: 4+ A? google.com. (28)
01:11:09.170110 IP 172.20.3.91.49950 > 10.68.0.2.domain: 5+ PTR? e.0.0.2.0.0.0.0.0.0.0.0.0.0.0.0.0.0.8.0.8.0.0.4.0.0.8.6.4.0.4.2.ip6.arpa. (90)
01:11:09.199710 IP 172.20.3.91.38714 > 10.68.0.2.domain: 6+ PTR? 238.200.58.216.in-addr.arpa. (45)
```

> nsenter -t <Pid> 进入container的network namespace 进行tcpdump，抓取UDP协议及dns查询的53端口

可以看出，容器中的DNS报文，没有之前多余的DNS查询，直接查询绝对域名`google.com.`

`coredns-configmap.yaml`

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health
        kubernetes cluster.local in-addr.arpa ip6.arpa {
          pods insecure
          upstream
          fallthrough in-addr.arpa ip6.arpa
        }
        prometheus :9153
        forward . /etc/resolv.conf
        cache 30
        loop
        reload
        loadbalance
        autopath @kubernetes
    }
```

```
cache [TTL] [ZONES...]  
```

`cache 30` 表示将记录缓存30秒 缓存的**ZONES**区域如果为空，则为所有区域启用缓存

