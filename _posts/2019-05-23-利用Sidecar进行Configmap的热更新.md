---
title: 利用Sidecar进行Configmap的热更新
date: 2019-05-23 22:53:46
tags: kubernetes
---

### ConfigMap概览

**ConfigMap API**资源用来保存**key-value pair**配置数据，这个数据可以在**pods**里使用，或者被用来为像**controller**一样的系统组件存储配置数据。虽然ConfigMap跟[Secrets](https://kubernetes.io/docs/user-guide/secrets/)类似，但是ConfigMap更方便的处理不含敏感信息的字符串。

ConfigMap API给我们提供了向容器中注入配置信息的机制，ConfigMap可以被用来保存单个属性，也可以用来保存整个配置文件或者JSON二进制大对象。

`example-configmap.yaml`

```yaml
kind: ConfigMap 
apiVersion: v1 
metadata:
  name: example-configmap 
data:
  # Configuration values can be set as key-value properties
  database: mongodb
  database_uri: mongodb://localhost:27017
  
  # Or set as complete file contents (even JSON!)
  keys: | 
    image.public.key=771 
    rsa.public.key=42
```

### 创建Configmap

```yaml
kubectel apply -f example-configmap.yaml
```



### 设置环境变量中使用ConfigMap

`pod-env-var.yaml`

```yaml
kind: Pod 
apiVersion: v1 
metadata:
  name: pod-env-var 
spec:
  containers:
    - name: env-var-configmap
      image: nginx:1.10
      envFrom:
        - configMapRef:
            name: example-configmap
```



### 进入Pod查看环境变量

```shell
$ kubectl apply -f pod-env-var.yaml
 pod "pod-env-var" created
$ kubectl exec -it pod-env-var sh
# env
# ...
KUBERNETES_SERVICE_PORT=443
KUBERNETES_PORT=tcp://10.68.0.1:443
database=mongodb
HOSTNAME=pod-env-var
database_uri=mongodb://localhost:27017
keys=image.public.key=771 
rsa.public.key=42
# ...

```

Configmap 对象是支持热更新的,可以参考之前的文章[在Kubernetes上ConfigMap热更新测试](https://jaeger.tk/2019/01/03/%E5%9C%A8Kubernetes%E4%B8%8AConfigMap%E7%83%AD%E6%9B%B4%E6%96%B0%E6%B5%8B%E8%AF%95/)，对 Configmap 的变更，会同时反应到加载该 Configmap 的 Pod 之中。但美中不足的是，很多应用都不会检测配置文件的更新，因此就算是通过对 Configmap 的变更，完成了配置文件的修改，应用还是无法做出即时的响应的。可以在外部进行滚动更新；或者改写业务容器，监控文件变化之后重新启动业务进程。



在 Kubernetes 1.10 中新增的 Pod 内共享进程命名空间`ShareProcessNamespace`的功能，并且Kubernetes 1.12 成为beta状态，给这个问题带来了一点新思路：做一个 Sidecar 用于对配置文件进行监控，发现文件变化之后，发送重新载入的信号给业务进程，要求业务进程自行刷新。这样就无需对业务容器所在镜像进行修改了。

> 这样的方式也有其局限性，需要业务进程支持重载信号

####  以Nginx 为例子实现

`nginx-cm.yaml`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-conf
data:
  nginx.conf: |
    user nginx;
    worker_processes  4;
    error_log  /var/log/nginx/error.log;
    events {
      worker_connections  65535;
    }
    http {
      log_format  main
              'remote_addr:$remote_addr\t'
              'time_local:$time_local\t'
              'method:$request_method\t'
              'uri:$request_uri\t'
              'host:$host\t'
              'status:$status\t'
              'bytes_sent:$body_bytes_sent\t'
              'referer:$http_referer\t'
              'useragent:$http_user_agent\t'
              'forwardedfor:$http_x_forwarded_for\t'
              'request_time:$request_time';
      access_log	/var/log/nginx/access.log main;
      server {
          listen       80;
          server_name  localhost;
          location / {
              root   html;
              index  index.html index.htm;
          }
      }
      include /etc/nginx/conf.d/*.conf;
    }
```

```
$ kubectl app -f nginx-cm.yaml
```

#### 创建sidecar容器镜像

`docker-entrypoint.sh`

```shell
#!/bin/sh
while true
do
	REAL=`readlink -f ${FILE}`
	inotifywait --exclude .swp -e create -e modify -e delete -e move "${REAL}"
	PID=`pgrep ${PROCESS} | head -n 1`
        case "$SIGNAL" in
	    HUP)
	        echo "Reloading Nginx Configuration"
		;;
	    USR1)
		echo "logrotate nginx log"
		;;
	    QUIT)
		echo "nginx service stop"
		;;
	    *)
		echo "unsupport signal：$SIGNAL ..."
		exit 2
	esac
	kill "-${SIGNAL}" "${PID}"
done
```

##### 构建sidecar镜像

```
FROM alpine

ENV FILE="/tmp" PROCESS="nginx" SIGNAL="HUP"

RUN apk add --update inotify-tools

COPY docker-entrypoint.sh /

ENTRYPOINT ["/docker-entrypoint.sh"]
```

##### 创建Nginx负载

```yaml
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: nginx
    spec:
      shareProcessNamespace: true # enable shareProcessNamespace   
      containers:
      - name: nginx
        image: nginx:1.10
        ports:
        - containerPort: 80
        volumeMounts:
        - mountPath: /etc/nginx # mount nginx-conf volumn to /etc/nginx
          readOnly: true
          name: nginx-conf
        - mountPath: /var/log/nginx
          name: log
      - name: inotify-nginx
        image: 314315960/inotify-nginx:latest
        imagePullPolicy: Always
        securityContext:
          capabilities:
            add:
            - SYS_PTRACE
        volumeMounts:
          - name: nginx-conf
            mountPath: /etc/nginx
        env:
          - name: FILE
            value: "/etc/nginx/nginx.conf"
          - name: PROCESS
            value: "nginx"
          - name: SIGNAL
            value: "HUP"
      volumes:
      - name: nginx-conf
        configMap:
          name: nginx-conf # place ConfigMap `nginx-conf` on /etc/nginx
          items:
            - key: nginx.conf
              path: nginx.conf
      - name: log
        emptyDir: {}
```

这个yaml文件：

- 在 `template.spec` 中加入了 `shareProcessNamespace: true`，表示启用进行命名空间共享功能；

- 新增一个的 Sidecar 容器inotify-nginx；

- Nginx和 Sidecar 共享来自同一个 Configmap 的配置文件，根据加载情况为 Sidecar 定义了环境变量。

### 测试

```shell
root@k8s-master-1:~/k8s_manifests/configmap-rot-reload# kubectl apply -f nginx-cm.yaml 
configmap/nginx-conf created

root@k8s-master-1:~/k8s_manifests/configmap-rot-reload# kubectl apply -f nginx-deployment.yaml 
deployment.apps/nginx created

# 修改configmap内容 会触发inotifywait 
root@k8s-master-1:~/k8s_manifests/configmap-rot-reload# kubectl edit cm nginx-conf
configmap/nginx-conf edited

```

##### 日志输出

```
root@k8s-master-1:~/k8s_manifests/configmap-rot-reload# kubectl  logs -f nginx-7464cd899-jfbrf -c inotify-nginx 
Setting up watches.
Watches established.
Reloading Nginx Configuration
Setting up watches.
Watches established.
....
```

Nginx 收到`HUP`信号 对配置文件做了重新载入

### 总结 

对于Nginx、Gunicorn、HA-Proxy 等都可以使用这种支持信号控制的方式来完成配置热更新工作。



参考文档

https://linux.cn/article-9141-1.html
https://blog.fleeto.us/post/refresh-cm-with-signal/
https://k8smeetup.github.io/docs/tasks/configure-pod-container/configure-pod-configmap/
https://www.kancloud.cn/curder/nginx/96674
https://linux.die.net/man/1/inotifywait


