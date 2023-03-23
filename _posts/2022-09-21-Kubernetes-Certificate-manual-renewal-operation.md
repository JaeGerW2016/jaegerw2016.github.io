---
layout:     post
title:     kubernetes证书手动续期操作
subtitle:  Kubernetes Certificate Manual Renewal Operation
date:       2022-09-21
author:     J
catalog: true
tags:
    - Kubernetes
---

Kubernetes证书手动续期操作


# 背景
众所周知Kubernetes版本在持续更新迭代，鼓励持续更新到最新版本，证书会有个1年的时间限制，但是生产环境，非必要不会去轻易升级Kubernetes集群版本，所以就需要手动去更新证书


## 官方详细文档
[使用 kubeadm 进行证书管理](https://kubernetes.io/zh-cn/docs/tasks/administer-cluster/kubeadm/kubeadm-certs/)


> Tips：修改kubeadm源码 修改  kubeadm的cmd包路径 cmd/kubeadm/app/constants/constants.go中 CertificateValidity = time.Hour * 24 * 365 这个常量 再编译kubeadm

可以参考[kubeadm-certs](https://github.com/lework/kubeadm-certs)项目维护证书有效期设置为10年的kubeadm分支

## 手动更新证书操作

首先需要确认Kubernetes集群是通过`kubeadm`创建的，然后确认`kube-controller-manager`是否启用以下参数
- `--cluster-signing-cert-file`
- `--cluster-signing-key-file`

```shell
root@node1:~# cat /etc/kubernetes/manifests/kube-controller-manager.yaml | grep cluster-signing-cert-file
    - --cluster-signing-cert-file=/etc/kubernetes/ssl/ca.crt
root@node1:~# cat /etc/kubernetes/manifests/kube-controller-manager.yaml | grep cluster-signing-key-file
    - --cluster-signing-key-file=/etc/kubernetes/ssl/ca.key

```

### 检查当前证书时间
```shell
root@node1:~# kubeadm certs check-expiration
[check-expiration] Reading configuration from the cluster...
[check-expiration] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
W0921 15:23:19.822889 2113045 utils.go:69] The recommended value for "clusterDNS" in "KubeletConfiguration" is: [10.233.0.10]; the provided value is: [169.254.25.10]

CERTIFICATE                EXPIRES                  RESIDUAL TIME   CERTIFICATE AUTHORITY   EXTERNALLY MANAGED
admin.conf                 Sep 14, 2023 12:25 UTC   358d            ca                      no      
apiserver                  Sep 14, 2023 12:24 UTC   358d            ca                      no      
apiserver-kubelet-client   Sep 14, 2023 12:24 UTC   358d            ca                      no      
controller-manager.conf    Sep 14, 2023 12:25 UTC   358d            ca                      no      
front-proxy-client         Sep 14, 2023 12:24 UTC   358d            front-proxy-ca          no      
scheduler.conf             Sep 14, 2023 12:25 UTC   358d            ca                      no      

CERTIFICATE AUTHORITY   EXPIRES                  RESIDUAL TIME   EXTERNALLY MANAGED
ca                      Sep 06, 2032 07:17 UTC   9y              no      
front-proxy-ca          Sep 06, 2032 07:17 UTC   9y              no  

```
可以看出当前RESIDUAL TIME是358d

### 续签证书

```shell
root@node1:~# kubeadm  certs renew admin.conf  &
kubeadm  certs renew apiserver  &
kubeadm  certs renew apiserver-kubelet-client  &
kubeadm  certs renew controller-manager.conf  &
kubeadm  certs renew front-proxy-client  &
kubeadm  certs renew scheduler.conf  &
[1] 2114553
[2] 2114554
[3] 2114555
[4] 2114556
[5] 2114557
[6] 2114558
root@node1:~# [renew] Reading configuration from the cluster...
[renew] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
[renew] Reading configuration from the cluster...
[renew] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
[renew] Reading configuration from the cluster...
[renew] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
W0921 15:30:09.580080 2114555 utils.go:69] The recommended value for "clusterDNS" in "KubeletConfiguration" is: [10.233.0.10]; the provided value is: [169.254.25.10]

[renew] Reading configuration from the cluster...
[renew] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
[renew] Reading configuration from the cluster...
[renew] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
[renew] Reading configuration from the cluster...
[renew] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
W0921 15:30:09.787915 2114553 utils.go:69] The recommended value for "clusterDNS" in "KubeletConfiguration" is: [10.233.0.10]; the provided value is: [169.254.25.10]
W0921 15:30:09.809999 2114558 utils.go:69] The recommended value for "clusterDNS" in "KubeletConfiguration" is: [10.233.0.10]; the provided value is: [169.254.25.10]


W0921 15:30:09.953495 2114557 utils.go:69] The recommended value for "clusterDNS" in "KubeletConfiguration" is: [10.233.0.10]; the provided value is: [169.254.25.10]

W0921 15:30:10.024681 2114554 utils.go:69] The recommended value for "clusterDNS" in "KubeletConfiguration" is: [10.233.0.10]; the provided value is: [169.254.25.10]

W0921 15:30:10.156098 2114556 utils.go:69] The recommended value for "clusterDNS" in "KubeletConfiguration" is: [10.233.0.10]; the provided value is: [169.254.25.10]
certificate for serving the Kubernetes API renewed

certificate embedded in the kubeconfig file for the admin to use and for kubeadm itself renewed
certificate for the API server to connect to kubelet renewed
certificate for the front proxy client renewed
certificate embedded in the kubeconfig file for the scheduler manager to use renewed
certificate embedded in the kubeconfig file for the controller manager to use renewed

[1]   Done                    kubeadm certs renew admin.conf
[2]   Done                    kubeadm certs renew apiserver
[3]   Done                    kubeadm certs renew apiserver-kubelet-client
[4]   Done                    kubeadm certs renew controller-manager.conf
[5]-  Done                    kubeadm certs renew front-proxy-client
[6]+  Done                    kubeadm certs renew scheduler.conf

```

### 查看当前证书时间

```shell
root@node1:~# kubeadm certs check-expiration
[check-expiration] Reading configuration from the cluster...
[check-expiration] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
W0921 15:31:05.475481 2114763 utils.go:69] The recommended value for "clusterDNS" in "KubeletConfiguration" is: [10.233.0.10]; the provided value is: [169.254.25.10]

CERTIFICATE                EXPIRES                  RESIDUAL TIME   CERTIFICATE AUTHORITY   EXTERNALLY MANAGED
admin.conf                 Sep 21, 2023 07:30 UTC   364d            ca                      no      
apiserver                  Sep 21, 2023 07:30 UTC   364d            ca                      no      
apiserver-kubelet-client   Sep 21, 2023 07:30 UTC   364d            ca                      no      
controller-manager.conf    Sep 21, 2023 07:30 UTC   364d            ca                      no      
front-proxy-client         Sep 21, 2023 07:30 UTC   364d            front-proxy-ca          no      
scheduler.conf             Sep 21, 2023 07:30 UTC   364d            ca                      no      

CERTIFICATE AUTHORITY   EXPIRES                  RESIDUAL TIME   EXTERNALLY MANAGED
ca                      Sep 06, 2032 07:17 UTC   9y              no      
front-proxy-ca          Sep 06, 2032 07:17 UTC   9y              no

```
可以看出证书的RESIDUAL TIME是364d，说明证书renew成功

### 参考文档
https://kubernetes.io/zh-cn/docs/tasks/administer-cluster/kubeadm/kubeadm-certs/
https://github.com/kubernetes/website/pull/26851
