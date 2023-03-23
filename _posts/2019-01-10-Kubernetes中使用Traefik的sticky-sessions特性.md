---
title: Kubernetes中使用Traefik的sticky sessions特性
date: 2019-01-10 02:11:03
tags: kubernetes
---


### 背景
在使用Nginx作为前端页面的反向代理时，有时我们需要在服务端为用户的本次访问保存一些临时状态，这种临时状态通常被称为 `session`。当访问压力增大时，常用的办法是开启多个服务端实例，然后使用Nginx一类的反向代理服务器进行负载均衡。然而对于依赖会话与用户进行交互的页面来说，由于负载均衡可能会将同一用户的两次访问分发到不同的服务端上，这样会话就无法正常运作了。而解决这个问题有最常用的两种方法，
- 一种是在应用层面上作修改，以支持会话共享。
- 一种方式则是使用会话粘粘：在客户端第一次发起请求时，反向代理为客户端分配一个服务端，并且将该服务端的地址以SetCookie的形式发送给客户端，这样客户端下一次访问该反向代理时，便会带着这个cookie，里面包含了上一次反向代理分配给该客户端的服务端信息。
>在Nginx中，这种机制是通过一个名为Sticky的插件实现的。而Traefik则集成了与Nginx的sticky sessions相同功能，并且可以在Kubernetes中方便的开启和配置该特性。


关于Traefik 服务发现部署部分可以参照之前的文章
[Kubernetes 服务发布之traefik ingress](https://www.jianshu.com/p/67118d1b9fa5)

#### kubernetes 中service开启Sticky
Kubernetes中可以通过为指定的service对象或ingress对象声明annotation来为ingress controller做额外的详细配置。例如，如果要开启sticky，只需要在想要开启sticky的服务端对应的service上添加以下的annotation即可。
`示例配置`
```
apiVersion: v1
kind: Service
metadata:
  name: traefik-web-ui
  namespace: kube-system
  annotations:
    traefik.ingress.kubernetes.io/affinity: "true"
    traefik.ingress.kubernetes.io/session-cookie-name: "traefik-cookie"
spec:
  selector:
    k8s-app: traefik-ingress-lb
  ports:
  - name: web
    port: 80
    targetPort: 8080
```
[Traefik 官方文档](https://docs.traefik.io/configuration/backends/kubernetes/)

![image.png](https://upload-images.jianshu.io/upload_images/3481257-6ea901d14c0726bf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

