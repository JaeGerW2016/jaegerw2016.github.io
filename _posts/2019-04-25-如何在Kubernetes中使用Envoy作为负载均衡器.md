---
title: 如何在Kubernetes中使用Envoy作为负载均衡器
date: 2019-04-25 00:58:45
tags: Envoy,kubernetes
---
在如今高度分布式的世界中，单主机架构越来越多地被多个更小的微服务架构所取代（无论好坏），代理和负载均衡技术似乎有了复兴。除之前经典技术了之外，近年来还出现了几种新的代理技术，这些技术以各种技术实现，通过不同的功能推广自己，例如轻松集成到某些云提供商（“云原生”），高性能和低内存占用，或动态配置。
可以说两种最流行的“经典”代理技术是[NGINX](https://www.nginx.com/)（C）和[HAProxy](http://www.haproxy.org/)（C），而其中包含的一些新生力量是[Zuul](https://github.com/Netflix/zuul)（Java），[Linkerd](https://linkerd.io/)（Rust），[Traefik](https://traefik.io/)（Go），[Caddy](https://github.com/mholt/caddy)（Go）和[Envoy](https://www.envoyproxy.io/)（C ++）。
所有这些技术都具有不同的功能集，并且针对某些特定方案或托管环境（例如，Linkerd经过微调以便在Kubernetes中使用）。
在这篇文章中，我不打算对这些进行比较，而只关注一个特定的场景：如何使用Envoy作为Kubernetes中运行的服务的负载均衡器。
[Envoy](https://www.envoyproxy.io/)是一个“高性能C ++分布式代理”，最初在Lyft实现，但从那时起就获得了广泛采用。它性能高，资源占用少，支持“控制平面”API管理的动态配置，并提供一些高级功能，如各种负载平衡算法，速率限制，熔断和镜像。
出于多种原因，我选择Envoy作为负载均衡器代理。
- 除了能够使用控制平面API动态控制外，它还支持简单，硬编码的基于YAML的配置，这对我的目的很方便，并且易于上手。
- 它内置了对其调用的服务发现技术的支持，该技术STRICT_DNS建立在查询DNS记录的基础上，并期望看到具有IP地址的A记录，用于上游集群的每个节点。这使得Kubernetes的无头服务变得简单易用。
- 它支持各种负载平衡算法，其中包括:`轮询round_robin`   `最小请求least_request`  `随机random` 。

在开始使用Envoy之前，我通过service类型的对象访问Kubernetes中的服务LoadBalancer，这是从Kubernetes外部访问服务的一种非常典型的方式。负载均衡器服务的确切工作方式取决于托管环境。 如果它首先支持它。我使用的是Google Kubernetes Engine，其中每个负载均衡器服务都映射到TCP级别的Google Cloud负载均衡器，该负载均衡器仅支持`轮询round_robin`算法。


### 1.为应用程序创建[Headless Service](https://kubernetes.io/docs/concepts/services-networking/service/#headless-services)
在Kubernetes中有一种称为[Headless Service](https://kubernetes.io/docs/concepts/services-networking/service/#headless-services)的特定服务，恰好与Envoy的`STRICT_DNS`服务发现模式一起使用非常方便。

Headless Service不提供单个IP和负载平衡到底层pod，而是它只有DNS配置，它为我们提供A记录，其中包含与标签选择器匹配的所有pod的pod的IP地址。  
此服务类型旨在用于我们希望实现负载平衡以及自己维护与上游pod的连接的场景，这正是我们可以使用Envoy执行的操作。
我们可以通过设置`.spec.clusterIP`字段来创建Headless Service `"None"`。因此，假设我们的应用程序pod具有app标签myapp，我们可以使用以下yaml创建Headless Service。
```
apiVersion: v1
kind: Service
metadata:
  name: myapp
  namespace: default
spec:
  clusterIP: None
  ports:
  - name: web
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: myapp
  sessionAffinity: None
  type: ClusterIP
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata: 
  name: myapp
  namespace: default
spec:
  replicas: 3
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: nginx:1.10
        ports:
        - containerPort: 80
```
myapp运行情况
```
root@k8s-master-1:~# kubectl get pod,svc -n default -o wide
NAME                               READY   STATUS    RESTARTS   AGE    IP             NODE           NOMINATED NODE   READINESS GATES
pod/myapp-c78bcd8fb-fkkfh          1/1     Running   0          31m    172.20.3.103   192.168.2.13   <none>           <none>
pod/myapp-c78bcd8fb-plnkt          1/1     Running   0          126m   172.20.2.62    192.168.2.12   <none>           <none>
pod/myapp-c78bcd8fb-s87sv          1/1     Running   0          31m    172.20.1.207   192.168.2.11   <none>           <none>
pod/myapp-envoy-696b6d764d-nm6lv   1/1     Running   1          18h    172.20.3.101   192.168.2.13   <none>           <none>

NAME                  TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)           AGE    SELECTOR
service/kubernetes    ClusterIP   10.68.0.1      <none>        443/TCP           60d    <none>
service/myapp         ClusterIP   None           <none>        80/TCP            122m   app=myapp
service/myapp-envoy   ClusterIP   10.68.246.10   <none>        80/TCP,9901/TCP   18h    app=myapp-envoy

```
现在，如果我们检查Kubernetes集群内的服务的DNS记录，我们将看到具有IP地址的单独A记录。如果我们有3个pod，我们会看到类似于此的DNS摘要。
```
root@k8s-master-1:~# kubectl run --attach busybox --rm --image=busybox:1.27 --restart=Never -- sh -c "sleep 4 && nslookup myapp.default"
If you don't see a command prompt, try pressing enter.
Server:    10.68.0.2
Address 1: 10.68.0.2 kube-dns.kube-system.svc.cluster.local

Name:      myapp.default
Address 1: 172.20.2.62 172-20-2-62.myapp.default.svc.cluster.local
Address 2: 172.20.1.207 172-20-1-207.myapp.default.svc.cluster.local
Address 3: 172.20.3.103 172-20-3-103.myapp.default.svc.cluster.local
pod "busybox" deleted

```
Envoy的`STRICT_DNS`的服务发现工作原理是，它维护的DNS服务器返回的所有A记录的IP地址，并每两秒钟刷新组IP地址。

### 2.创建Envoy 镜像
在不提供动态API形式的控制平面的情况下使用Envoy的最简单方法是将硬编码配置添加到静态yaml文件中。
以下是对域名`myapp`给出的IP地址进行负载均衡的基本配置。
#### envoy.yaml
```
admin:
  access_log_path: /tmp/admin_access.log
  address:
    socket_address: { address: 0.0.0.0, port_value: 9901 }

static_resources:
  listeners:
  - name: listener_0
    address:
      socket_address: { address: 0.0.0.0, port_value: 80 }
    filter_chains:
    - filters:
      - name: envoy.http_connection_manager
        config:
          stat_prefix: ingress_http
          route_config:
            name: local_route
            virtual_hosts:
            - name: local_service
              domains: ["*"]
              routes:
              - match: { prefix: "/" }
                route: { host_rewrite: myapp, cluster: myapp_cluster, timeout: 60s }
          http_filters:
          - name: envoy.router
  clusters:
  - name: myapp_cluster
    connect_timeout: 0.25s
    type: STRICT_DNS
    dns_lookup_family: V4_ONLY
    lb_policy: ${ENVOY_LB_ALG}
    hosts: [{ socket_address: { address: ${SERVICE_NAME}, port_value: 80 }}]

```

#### docker-entrypoint.sh
```
#!/bin/sh
set -e

echo "Generating envoy.yaml config file..."
cat /tmpl/envoy.yaml.tmpl | envsubst \$ENVOY_LB_ALG,\$SERVICE_NAME > /etc/envoy.yaml

echo "Starting Envoy..."
/usr/local/bin/envoy -c /etc/envoy.yaml
```

#### Dockerfile
```
FROM envoyproxy/envoy:latest

COPY envoy.yaml /tmpl/envoy.yaml.tmpl
COPY docker-entrypoint.sh /

RUN chmod 500 /docker-entrypoint.sh

RUN apt-get update && \
    apt-get install gettext -y && \
    rm -rf /var/lib/apt/list/* && \
    rm -rf /var/cache/apk/*

ENTRYPOINT ["/docker-entrypoint.sh"]
```

### 3.创建Envoy的Deployment
```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: myapp-envoy
  labels:
    app: myapp-envoy
spec:
  selector:
    matchLabels:
      app: myapp-envoy
  template:
    metadata:
      labels:
        app: myapp-envoy
    spec:
      containers:
      - name: myapp-envoy
        image: 314315960/myapp-envoyproxy:latest
        imagePullPolicy: IfNotPresent
        env:
        - name: "ENVOY_LB_ALG"
          value: "LEAST_REQUEST"
        - name: "SERVICE_NAME"
          value: "myapp"
        ports:
        - name: http
          containerPort: 80
        - name: envoy-admin
          containerPort: 9901

```
应用此yaml后，Envoy代理应该可以运行，您可以通过将请求发送到Envoy服务的主端口来访问底层服务。

在此示例中，我仅添加了ClusterIP类型的服务，但如果要从群集外部访问代理，还可以使用LoadBalancer服务或Ingress对象。

![image.png](https://upload-images.jianshu.io/upload_images/3481257-1fe053801c15e196.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 4.Troubleshooting
在Envoy配置文件中，您可以看到`admin`部分，它配置Envoy的管理端点。这可用于检查有关代理的各种诊断信息。
一些有用的节点：
- `/config_dump`: 打印代理的完整配置，这有助于验证pod上是否有正确的配置
- `/clusters`: 显示Envoy发现的所有上游节点，以及为每个节点处理的请求数。这对于检查负载平衡算法是否正常工作非常有用。
![](https://upload-images.jianshu.io/upload_images/3481257-ed332994b2ec85c6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](https://upload-images.jianshu.io/upload_images/3481257-786c16ef8478286c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![image.png](https://upload-images.jianshu.io/upload_images/3481257-b0ce8ebc94daeda6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 5.验证
#### myapp-ingress.yaml
```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: myapp-ingress
  namespace: default
spec:
  rules:
  - host: myapp.k8s.io
    http:
      paths:
      - path: /
        backend:
          serviceName: myapp
          servicePort: web
```

#### myapp-envoy-ingress.yaml
```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: myapp-envoy-ingress
  namespace: default
spec:
  rules:
  - host: myapp-envoy.k8s.io
    http:
      paths:
      - path: /
        backend:
          serviceName: myapp-envoy
          servicePort: 80

```
![](https://upload-images.jianshu.io/upload_images/3481257-2be74268de462f8f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![](https://upload-images.jianshu.io/upload_images/3481257-61b41efb0934a070.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

参考文档：
https://jimmysong.io/posts/envoy-archiecture-and-terminology/
https://blog.markvincze.com/how-to-use-envoy-as-a-load-balancer-in-kubernetes/
https://kubernetes.io/docs/concepts/services-networking/ingress/
