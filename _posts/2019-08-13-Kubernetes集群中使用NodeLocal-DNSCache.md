---
title: Kubernetes集群中使用NodeLocal DNSCache
date: 2019-08-13 11:23
tag: kubernetes
---

### 介绍

NodeLocal DNSCache通过在群集节点上运行DaemonSet的Pod来做dns缓存代理提高群集DNS性能。

在没有NodeLocal DNSCache的架构中，集群中的Pod默认ClusterFirst DNS模式中的可以访问kube-dns serviceIP以进行DNS查询。通过kube-proxy添加的iptables规则将其转换为kube-dns / CoreDNS端点。

在使用NodeLocal DNSCache的架构中，Pods将访问Pod所在的节点上运行的dns缓存代理，从而避免iptables DNAT和conntrack。Pod在查询集群中主机名的缓存未命中情况下 DNS缓存代理将查询kube-dns服务（默认cluster_domain为cluster.local后缀）

### 用NodeLocal DNSCache的意图

- 在没有NodeLocal DNSCache的架构中，CoreDNS 一般以deployment的形式部署在集群中，如果Pod所在的Node上没有kube-dns / CoreDNS实例，则具有最高DNS QPS的Pod可能必须跨Node访问CoreDNS实例。对于NodeLocal DNSCache有助于改善此类情况下的延迟。
- 跳过iptables DNAT和conntrack将有助于减少[conntrack竞争](https://github.com/kubernetes/kubernetes/issues/56903)并避免UDP DNS条目填满conntrack表
- 从本地缓存代理到kube-dns服务的连接可以升级到TCP。连接关闭时TCP conntrack条目将被删除，而UDP条目必须超时（[默认](https://www.kernel.org/doc/Documentation/networking/nf_conntrack-sysctl.txt) `nf_conntrack_udp_timeout`为30秒）
- 将DNS查询从UDP升级到TCP将减少由于丢弃的UDP数据包导致的尾部延迟和DNS超时，通常最多30秒（3次重试+ 10秒超时）。由于nodelocal缓存侦听UDP DNS查询，因此不需要更改应用程序。
- 在节点级别对dns请求的度量标准和可见性。
- 可以重新启用负缓存，从而减少对kube-dns服务的查询次数。

#### Nodelocal DNSCache流程

此图显示了NodeLocal DNSCahe如何处理DNS查询。

![](https://i.loli.net/2019/08/13/28NFbv5IpH3eimD.jpg)

## 配置

```shell
curl -s -O https://raw.githubusercontent.com/kubernetes/kubernetes/master/cluster/addons/dns/nodelocaldns/nodelocaldns.yaml

#dnsPolicy: Default  # Don't use cluster DNS.
sed -i -e "s/__PILLAR__DNS__DOMAIN__/${KUBE_DNS_NAME:-cluster.local.}/g" nodelocaldns.yaml
sed -i -e "s/__PILLAR__DNS__SERVER__/${KUBE_DNS_SERVER_IP:-10.68.0.2}/g" nodelocaldns.yaml
sed -i -e "s/__PILLAR__LOCAL__DNS__/${KUBE_LOCAL_DNS_IP:-169.254.20.10}/g" nodelocaldns.yaml

kubectl create -f nodelocaldns.yaml


//修改kubelet --cluster-dns=169.254.20.10

sed -i -e "s/--cluster-dns=${KUBE_DNS_SERVER_IP:-10.68.0.2}/--cluster-dns=${KUBE_LOCAL_DNS_IP:-169.254.20.10}/g" /etc/systemd/system/kubelet.service
systemctl daemon-reload
systemctl restart kubelet.service

```

| k8s version | Feature support               |
| :---------- | :---------------------------- |
| 1.15        | Beta(Not enabled by default)  |
| 1.13        | Alpha(Not enabled by default) |



### 参考文档

https://kubernetes.io/docs/tasks/administer-cluster/nodelocaldns/

https://github.com/kubernetes/kubernetes/issues/56903

https://www.kernel.org/doc/Documentation/networking/nf_conntrack-sysctl.txt