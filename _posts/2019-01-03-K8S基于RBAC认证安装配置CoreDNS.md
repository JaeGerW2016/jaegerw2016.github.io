---
title: K8S基于RBAC认证安装配置CoreDNS
date: 2019-01-03 02:36:44
tags:
---
###文档说明
实验环境：kubernetes Version v1.9.6
网络CNI：fannel
存储CSI: NFS Dynamic Class
DNS: kube-DNS 替换CoreDNS

####背景
CoreDNS是一个Go语言实现的链式插件DNS服务端，是CNCF成员，是一个高性能、易扩展的DNS服务端。可以很方便的部署在k8s集群中，用来代替kube-dns。
>Our goal is to make CoreDNS the cloud-native DNS server and service discovery solution.

部署 CoreDNS 需要使用到官方提供的两个文件 [deploy.sh](https://github.com/coredns/deployment/blob/master/kubernetes/deploy.sh)和[coredns.yaml.sed](https://github.com/coredns/deployment/blob/master/kubernetes/coredns.yaml.sed)（这两个文件已经放入manifest的coredns目录中）

`deploy.sh` 是一个用于在已经运行kube-dns的集群中生成运行CoreDNS部署文件（manifest）的工具脚本。它使用 `coredns.yaml.sed`文件作为模板，创建一个ConfigMap和CoreDNS的deployment，然后更新集群中已有的kube-dns 服务的selector使用CoreDNS的deployment。重用已有的服务并不会在服务的请求中发生冲突。

`deploy.sh`文件并不会删除kube-dns的deployment或者replication controller。如果要删除kube-dns，你必须在部署CoreDNS后手动的删除kube-dns。

你需要仔细测试manifest文件，以确保它能够对你的集群正常运行。这依赖于你的怎样构建你的集群以及你正在运行的集群版本。 对manifest文件做一些修改是有比要的。

在最佳的案例场景中，使用CoreDNS替换Kube-DNS只需要使用下面的两个命令：

```
$ ./deploy.sh | kubectl apply -f -
$ kubectl delete --namespace=kube-system deployment kube-dns

```

注意：我们建议在部署CoreDNS后删除kube-dns。否则如果CoreDNS和kube-dns同时运行，服务查询可能会随机的在CoreDNS和kube-dns之间产生。

对于非RBAC部署，你需要编辑生成的结果yaml文件：

1.  从yaml文件的`Deployment`部分删除 `serviceAccountName: coredns`
2.  删除 `ServiceAccount`、 `ClusterRole`和 `ClusterRoleBinding` 部分

####替换过程中碰到的error及Q&A:

> Q:  plugin/loop: Seen “HINFO IN xxx.” more than twice, loop detected

```
修改 kubelet的配置文件，加上 --resovf-conf标志
--resolv-conf=/run/systemd/resolve/resolv.conf # Ubuntu 18.04 
--resolv-conf=/etc/resolv.conf #default
路径：/etc/systemd/system/kubelet.service 或者 /etc/systemd/system/kubelet.service.d  目录下的配置

systemctl daemon-reload && systemctl restart kubelet

[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=docker.service
Requires=docker.service

[Service]
WorkingDirectory=/var/lib/kubelet
#--pod-infra-container-image=registry.access.redhat.com/rhel7/pod-infrastructure:latest
ExecStart=/opt/kube/bin/kubelet \
  --address=192.168.2.11 \
  --hostname-override=192.168.2.11 \
  --pod-infra-container-image=mirrorgooglecontainers/pause-amd64:3.1 \
  --experimental-bootstrap-kubeconfig=/etc/kubernetes/bootstrap.kubeconfig \
  --kubeconfig=/etc/kubernetes/kubelet.kubeconfig \
  --cert-dir=/etc/kubernetes/ssl \
  --client-ca-file=/etc/kubernetes/ssl/ca.pem \
  --network-plugin=cni \
  --cni-conf-dir=/etc/cni/net.d \
  --cni-bin-dir=/opt/kube/bin \
  --cluster-dns=10.68.0.2 \
  --cluster-domain=cluster.local. \
  --hairpin-mode hairpin-veth \
  --allow-privileged=true \
  --fail-swap-on=false \
  --anonymous-auth=false \
  --logtostderr=true \
  --v=2 \
  --resolv-conf=/run/systemd/resolve/resolv.conf  #Ubuntu 18.04 DNS配置
#kubelet cAdvisor 默认在所有接口监听 4194 端口的请求, 以下iptables限制内网访问
ExecStartPost=/sbin/iptables -A INPUT -s 10.0.0.0/8 -p tcp --dport 4194 -j ACCEPT
ExecStartPost=/sbin/iptables -A INPUT -s 172.16.0.0/12 -p tcp --dport 4194 -j ACCEPT
ExecStartPost=/sbin/iptables -A INPUT -s 192.168.0.0/16 -p tcp --dport 4194 -j ACCEPT
ExecStartPost=/sbin/iptables -A INPUT -p tcp --dport 4194 -j DROP
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target

```


参考文档
https://jimmysong.io/kubernetes-handbook/practice/coredns.html
https://github.com/coredns/coredns/issues/2087
https://zhengyinyong.com/coredns-basis.html
http://johng.cn/minikube-fatal-pluginloop-seen/
https://kubernetes.io/docs/setup/independent/troubleshooting-kubeadm/#coredns-pods-have-crashloopbackoff-or-error-state

