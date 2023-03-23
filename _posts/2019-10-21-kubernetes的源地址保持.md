---
title: kubernetes的源地址保持sourceIP
date: 2019-10-21 16:53
tag: kubernetes
---

## Prerequisites

Kubernetes `Service` 定义了这样一种抽象：逻辑上的一组 `Pod`，一种可以访问它们的策略 —— 通常称为微服务。 这一组 `Pod` 能够被 `Service` 访问到，通常是通过 [selector](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/) （查看[下面](https://kubernetes.io/zh/docs/concepts/services-networking/service/#services-without-selectors)了解，为什么你可能需要没有 selector 的 `Service`）实现的。

kubernetes `service`借助iptables将一组POD抽象成可达的网络服务，并且由于kubernetes要保证`service`在任何node的可达性，所以使用iptables rule将所有后端POD组成一个基于概率访问的组合，使用iptables的SNAT和DNAT技术在不同POD之间进行基于概率的请求转发。这样的模式带来了一个副作用：经过SNAT操作之后，客户端的IP消失在了请求的接入node，应用程序POD只能看到node的IP，对于一些对源IP有需求的应用来讲，需要kubernetes提供解决这个问题的机制。而kubernetes的source preserve的功能(https://kubernetes.io/docs/tutorials/services/source-ip/)就是为解决这个问题而引入的。

## Source IP for Services with Type=ClusterIP

如果您在[iptables模式下](https://s0kubernetes0io.icopy.site/docs/concepts/services-networking/service/#proxy-mode-iptables)运行kube-proxy，则从群集内部发送到ClusterIP的数据包永远不会源NAT，这是Kubernetes 1.2以来的默认[模式](https://s0kubernetes0io.icopy.site/docs/concepts/services-networking/service/#proxy-mode-iptables) . Kube-proxy通过`proxyMode`端点公开其模式：

```console
kubectl get nodes
```

输出类似于以下内容：

```
NAME                           STATUS     ROLES    AGE     VERSION
kubernetes-node-6jst   Ready      <none>   2h      v1.13.0
kubernetes-node-cx31   Ready      <none>   2h      v1.13.0
kubernetes-node-jj1t   Ready      <none>   2h      v1.13.0
```

在节点之一上获取代理模式

```console
kubernetes-node-6jst $ curl localhost:10249/proxyMode
```

输出为：

```
iptables
```

您可以通过在源IP应用程序上创建服务来测试源IP保留：

```console
kubectl expose deployment source-ip-app --name=clusterip --port=80 --target-port=8080
```

输出为：

```
service/clusterip exposed
kubectl get svc clusterip
```

输出类似于：

```
NAME         TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)   AGE
clusterip    ClusterIP   10.0.170.92   <none>        80/TCP    51s
```

并从同一集群中的pod命中`ClusterIP` ：

```console
kubectl run busybox -it --image=busybox --restart=Never --rm
```

输出类似于以下内容：

```
Waiting for pod default/busybox to be running, status is Pending, pod ready: false
If you don't see a command prompt, try pressing enter.

# ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
3: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1460 qdisc noqueue
    link/ether 0a:58:0a:f4:03:08 brd ff:ff:ff:ff:ff:ff
    inet 10.244.3.8/24 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::188a:84ff:feb0:26a5/64 scope link
       valid_lft forever preferred_lft forever

# wget -qO - 10.0.170.92
CLIENT VALUES:
client_address=10.244.3.8
command=GET
...
```

无论客户端容器和服务器容器是在同一节点还是在不同节点中，client_address始终是客户端容器的IP地址.

## Source IP for Services with Type=NodePort

从Kubernetes 1.5开始，默认情况下发送到具有[Type = NodePort的](https://s0kubernetes0io.icopy.site/docs/concepts/services-networking/service/#nodeport)服务的数据包是源NAT. 您可以通过创建`NodePort`服务进行测试：

```console
kubectl expose deployment source-ip-app --name=nodeport --port=80 --target-port=8080 --type=NodePort
```

输出为：

```
service/nodeport exposed
NODEPORT=$(kubectl get -o jsonpath="{.spec.ports[0].nodePort}" services nodeport)
NODES=$(kubectl get nodes -o jsonpath='{ $.items[*].status.addresses[?(@.type=="ExternalIP")].address }')
```

如果您在cloudprovider上运行，则可能需要为上面报告的`nodes:nodeport`打开防火墙规则. 现在，您可以尝试通过上面分配的节点端口从群集外部访问服务.

```console
for node in $NODES; do curl -s $node:$NODEPORT | grep -i client_address; done
```

输出类似于：

```
client_address=10.180.1.1
client_address=10.240.0.5
client_address=10.240.0.3
```

请注意，这些不是正确的客户端IP，它们是群集内部IP. 这是发生了什么：

- 客户端将数据包发送到`node2:nodePort`
- `node2`将数据包中的源IP地址（SNAT）替换为其自己的IP地址
- `node2`用Pod IP替换数据包上的目标IP
- 数据包路由到节点1，然后路由到端点
- Pod的回复被路由回node2
- pod的回复发送回客户端

Visually:

```
          client
             \ ^
              \ \
               v \
   node 1 <--- node 2
    | ^   SNAT
    | |   --->
    v |
 endpoint
```

为了避免这种情况，Kubernetes具有保留客户端源IP [的功能（请在此处查看功能可用性）](https://s0kubernetes0io.icopy.site/docs/tasks/access-application-cluster/create-external-load-balancer/#preserving-the-client-source-ip) . 将`service.spec.externalTrafficPolicy`设置为`Local`值将仅将请求代理到本地终结点，而不将流量转发到其他节点，从而保留原始源IP地址. 如果没有本地端点，则将丢弃发送到该节点的数据包，因此您可以在任何数据包处理规则中依赖正确的source-ip，您可以应用将其直达端点的数据包.

如下设置`service.spec.externalTrafficPolicy`字段：

```console
kubectl patch svc nodeport -p '{"spec":{"externalTrafficPolicy":"Local"}}'
```

输出为：

```
service/nodeport patched
```

现在，重新运行测试：

```console
for node in $NODES; do curl --connect-timeout 1 -s $node:$NODEPORT | grep -i client_address; done
```

输出为：

```
client_address=104.132.1.79
```

请注意，只有端点Pod在其上运行的一个节点上，使用*正确的*客户端IP才能收到一个答复.

这是发生了什么：

- 客户端将数据包发送到没有任何端点的`node2:nodePort`
- 数据包被丢弃
- 客户端发送数据包到`node1:nodePort` ，里面*确实*有端点
- node1使用正确的源IP将数据包路由到端点

Visually:

```
        client
       ^ /   \
      / /     \
     / v       X
   node 1     node 2
    ^ |
    | |
    | v
 endpoint
```

## Source IP for Services with Type=LoadBalancer

从Kubernetes 1.5开始，默认情况下发送到[Type = LoadBalancer的](https://s0kubernetes0io.icopy.site/docs/concepts/services-networking/service/#loadbalancer) Services的数据包是源NAT，因为所有处于`Ready`状态的可调度Kubernetes节点均可进行负载平衡流量. 因此，如果数据包到达没有端点的节点，则系统将其代理到*具有*端点的节点，用该节点的IP替换数据包上的源IP（如上一节所述）.

您可以通过通过负载均衡器公开source-ip-app进行测试

```console
kubectl expose deployment source-ip-app --name=loadbalancer --port=80 --target-port=8080 --type=LoadBalancer
```

输出为：

```
service/loadbalancer exposed
```

服务的打印IP：

```console
kubectl get svc loadbalancer
```

输出类似于以下内容：

```
NAME           TYPE           CLUSTER-IP    EXTERNAL-IP       PORT(S)   AGE
loadbalancer   LoadBalancer   10.0.65.118   104.198.149.140   80/TCP    5m
curl 104.198.149.140
```

输出类似于以下内容：

```
CLIENT VALUES:
client_address=10.240.0.5
...
```

但是，如果您在Google Kubernetes Engine / GCE上运行，则将相同的`service.spec.externalTrafficPolicy`字段设置为`Local`通过故意使运行状况检查失败而将*没有* Service终结点的节点从符合负载平衡流量的节点列表中删除.

Visually:

```
                      client
                        |
                      lb VIP
                     / ^
                    v /
health check --->   node 1   node 2 <--- health check
        200  <---   ^ |             ---> 500
                    | V
                 endpoint
```

您可以通过设置注释进行测试：

```console
kubectl patch svc loadbalancer -p '{"spec":{"externalTrafficPolicy":"Local"}}'
```

您应该立即看到Kubernetes分配的`service.spec.healthCheckNodePort`字段：

```console
kubectl get svc loadbalancer -o yaml | grep -i healthCheckNodePort
```

输出类似于以下内容：

```
  healthCheckNodePort: 32122
```

`service.spec.healthCheckNodePort`字段指向`/healthz`运行状况检查的每个节点上的端口. 您可以对此进行测试：

```console
kubectl get pod -o wide -l run=source-ip-app
```

输出类似于以下内容：

```
NAME                            READY     STATUS    RESTARTS   AGE       IP             NODE
source-ip-app-826191075-qehz4   1/1       Running   0          20h       10.180.1.136   kubernetes-node-6jst
```

在不同节点上卷曲`/healthz`端点.

```console
kubernetes-node-6jst $ curl localhost:32122/healthz
```

输出类似于以下内容：

```
1 Service Endpoints found
kubernetes-node-jj1t $ curl localhost:32122/healthz
```

输出类似于以下内容：

```
No Service Endpoints Found
```

在主服务器上运行的服务控制器负责分配云负载均衡器，并且当这样做时，它还会在每个节点上分配指向此端口/路径的HTTP运行状况检查. 等待大约10秒钟，使没有端点的2个节点无法通过运行状况检查，然后卷曲lb ip：

```console
curl 104.198.149.140
```

输出类似于以下内容：

```
CLIENT VALUES:
client_address=104.132.1.79
...
```