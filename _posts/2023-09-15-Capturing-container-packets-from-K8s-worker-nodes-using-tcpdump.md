---
layout: mypost
title: 使用tcpdump从K8s工作节点捕获容器数据包
categories: [Kubernetes,tcpdump,Network]
---
这篇博文将向您展示如何从K8s Node节点捕获容器数据包。



解决网络问题，并且必须进入数据包详细信息级别进行分析。要捕获数据包，您通常会`tcpdump`在源IP或目标IP处运行，但是在容器化环境或者Kubernetes集群环境中，可能会变的稍微有些区别。

在开始这篇博文之前，默认是你已经了解并会使用`tcpdump`命令,并且在k8s的工作节点已经安装了`tcpdump工具`

```shell
root@node1:~# tcpdump --version
tcpdump version 4.99.0
libpcap version 1.10.0 (with TPACKET_V3)
OpenSSL 1.1.1k  25 Mar 2021
```

### 查找Pod所在Node节点

```shell
root@node1:~# kubectl get pod -owide
NAME                               READY   STATUS    RESTARTS        AGE     IP             NODE    NOMINATED NODE   READINESS GATES
demo-deployment-6b756cbbc4-gxkjt   1/1     Running   1 (3d18h ago)   4d15h   10.233.64.19   node1   <none>           <none>
demo-deployment-6b756cbbc4-mstjb   1/1     Running   1 (3d18h ago)   4d15h   10.233.66.78   node3   <none>           <none>
```

可以看到 `demo-deployment-6b756cbbc4-gxkjt` 这个Pod所在的Node节点就在`node1`上面，详细信息可以`kubectl describe pod demo-deployment-6b756cbbc4-gxkjt -n default` 查看Pod的描述信息


```yaml
root@node1:~# kubectl describe pod demo-deployment-6b756cbbc4-gxkjt -n default
Name:         demo-deployment-6b756cbbc4-gxkjt
Namespace:    default
Priority:     0
Node:         node1/192.168.2.101
Start Time:   Tue, 12 Sep 2023 19:21:09 +0800
Labels:       app=demo-app
              pod-template-hash=6b756cbbc4
Annotations:  <none>
Status:       Running
IP:           10.233.64.19
IPs:
  IP:           10.233.64.19
Controlled By:  ReplicaSet/demo-deployment-6b756cbbc4
Containers:
  nginx:
    Container ID:   containerd://2fcfb3d89740b336a2970bad9db4628e15f847f62146b2e85a5f536fbae9b933
    Image:          nginx:latest
    Image ID:       docker.io/library/nginx@sha256:6926dd802f40e5e7257fded83e0d8030039642e4e10c4a98a6478e9c6fe06153
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Wed, 13 Sep 2023 17:11:43 +0800
    Last State:     Terminated
      Reason:       Unknown
      Exit Code:    255
      Started:      Tue, 12 Sep 2023 19:21:39 +0800
      Finished:     Wed, 13 Sep 2023 17:03:56 +0800
    Ready:          True
    Restart Count:  1
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-zd6ck (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             True 
  ContainersReady   True 
  PodScheduled      True 
Volumes:
  kube-api-access-zd6ck:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:                      <none>
```

### 查找Node节点上的容器网络接口
通过`kubeclt exec -it `命令来进入Pod的Container查看容器网络接口

```shell
root@node1:~# kubectl exec -it demo-deployment-6b756cbbc4-gxkjt -n default -- /bin/bash
root@demo-deployment-6b756cbbc4-gxkjt:/# 
root@demo-deployment-6b756cbbc4-gxkjt:/# cat /sys/class/net/eth0/iflink
17
```
可以查到网络接口的index索引，我们可以通过 SSH 连接并运行来定位Node节点上的网络接口。
通过以下命名在Node主机上查找容器网络接口对应的虚拟网络接口。

`ip link | grep <network interface index>`

```shell
root@node1:~# ip link | grep 17
...
17: lxcea5374ac0442@if16: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
...

```

### 从主机网络接口捕获数据包
现在我们已经有了所有必要的信息，我们可以开始在Node节点上运行命令来捕获数据包。

`-w` 参数 导出到cap文件中，`ctrl+c` 随时停止捕获

`tcpdump -i <network interface>-w <file name>.pcap`

```shell
root@node1:~# tcpdump -i lxcea5374ac0442 -w network.cap
tcpdump: listening on lxcea5374ac0442, link-type EN10MB (Ethernet), snapshot length 262144 bytes


root@node1:~# kubectl exec -it demo-deployment-6b756cbbc4-mstjb -n default -- /bin/bash
root@demo-deployment-6b756cbbc4-mstjb:/# 
root@demo-deployment-6b756cbbc4-mstjb:/# 
root@demo-deployment-6b756cbbc4-mstjb:/# 
root@demo-deployment-6b756cbbc4-mstjb:/# 
root@demo-deployment-6b756cbbc4-mstjb:/# 
root@demo-deployment-6b756cbbc4-mstjb:/# curl 10.233.64.19
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
root@demo-deployment-6b756cbbc4-mstjb:/# 

#ctrl+c 停止
root@node1:~# tcpdump -i lxcea5374ac0442 -w network.cap
tcpdump: listening on lxcea5374ac0442, link-type EN10MB (Ethernet), snapshot length 262144 bytes
^C14 packets captured
14 packets received by filter
0 packets dropped by kernel
```

### 在Wireshark上分析和查看数据包

通过导出的cap的文件，用Wireshark软件打开，就可以查看捕获的数据包，至此知道如何从Node节点捕获容器数据包。

[![wireshark_pic_202309175a543b73d9a27a1c.md.png](https://youjb.com/images/2023/09/17/wireshark_pic_202309175a543b73d9a27a1c.md.png)](https://youjb.com/image/cH7)