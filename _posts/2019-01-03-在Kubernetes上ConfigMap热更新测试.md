---
title: 在Kubernetes上ConfigMap热更新测试
date: 2019-01-03 02:46:00
tags:
---

### 文档说明
实验环境：kubernetes Version v1.9.6
网络CNI：fannel
存储CSI: NFS Dynamic Class



>获取configmap中的环境变量方式有2种：
1、使用该 ConfigMap 挂载的 Env
2、使用该 ConfigMap 挂载的 Volume

##### 更新使用ConfigMap挂载的Env
`nginx-cm.yaml`
```
apiVersion: v1
kind: ConfigMap
metadata: 
  name: env-config
  namespace: default
data:
  log_level: INFO
```
`nginx-deployment.yaml`

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata: 
  name: my-nginx
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: my-nginx
    spec:
      containers:
      - name: my-nginx
        image: nginx:1.10
        ports:
        - containerPort: 80
        envFrom:
        - configMapRef:
            name: env-config
```
##### 验证
![1.png](https://upload-images.jianshu.io/upload_images/3481257-1fe745b5069b4ff7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


>##### 实践证明修改 ConfigMap 无法更新容器中已注入的环境变量信息

##### 更新使用ConfigMap挂载的Volume
`nginx-cm2.yaml`
```
apiVersion: v1
kind: ConfigMap
metadata: 
  name: special-config
  namespace: default
data:
  log_level: INFO
```
`nginx-deployment-v.yaml`
```
apiVersion: extensions/v1beta1
kind: Deployment
metadata: 
  name: my-nginx-2
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: my-nginx-2
    spec:
      containers:
      - name: my-nginx-2
        image: nginx:1.10
        ports:
        - containerPort: 80
        volumeMounts:
        - name: config-volume
          mountPath: /etc/config
      volumes:
        - name: config-volume
          configMap:
            name: special-config
```

### 验证
![image.png](https://upload-images.jianshu.io/upload_images/3481257-7b012002ab22a96c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![image.png](https://upload-images.jianshu.io/upload_images/3481257-cf486bec9951eb66.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![image.png](https://upload-images.jianshu.io/upload_images/3481257-4e5f5c0210ac410c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 总结

更新 ConfigMap 后：

*   使用该 ConfigMap 挂载的 Env **不会**同步更新
*   使用该 ConfigMap 挂载的 Volume 中的数据需要一段时间（实测大概10秒）才能同步更新

ENV 是在容器启动的时候注入的，启动之后 kubernetes 就不会再改变环境变量的值，且同一个 namespace 中的 pod 的环境变量是不断累加的，参考 [Kubernetes中的服务发现与docker容器间的环境变量传递源码探究](https://jimmysong.io/posts/exploring-kubernetes-env-with-docker/)。为了更新容器中使用 ConfigMap 挂载的配置，可以通过滚动更新 pod 的方式来强制重新挂载 ConfigMap，也可以在更新了 ConfigMap 后，先将副本数设置为 0，然后再扩容。


参考文档：
https://jimmysong.io/posts/kubernetes-configmap-hot-update/
