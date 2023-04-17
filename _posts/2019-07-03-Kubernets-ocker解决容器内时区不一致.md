---
title: Kubernets/Docker解决容器内时区不一致
date: 2019-07-03 17:54:00
tags: kubernetes
---

## 1、背景介绍

我们知道，使用 docker 容器启动服务后，如果使用默认`alpine`作为基础镜像，就会出现系统时区不一致的问题，因为默认 alpine`系统时间为 UTC 协调世界时 (Universal Time Coordinated)，一般本地所属时区为 CST（＋8 时区，上海时间），时间上刚好相差 8 个小时。这就导致了，我们服务启动后，获取系统时间来进行相关操作，例如存入数据库、时间换算、日志记录等，都会出现时间不一致的问题，所以很有必要解决掉容器内时区不统一的问题。

#### 现象

```shell
[root@localhost ~]# date
Wed Jul  3 17:15:31 CST 2019
[root@localhost ~]# docker run -it alpine sh
/ # date
Wed Jul  3 09:15:39 UTC 2019
/ # 

```

## Dockerfile 中处理

可以直接修改 `Dockerfile`，在构建系统基础镜像或者基于基础镜像再次构建业务镜像时，添加时区修改配置即可

```dockerfile
FROM alpine:latest

RUN apk add -U tzdata \
&& rm -f /etc/localtime \
&& ln -sv /usr/share/zoneinfo/Asia/Shanghai /etc/localtime \
&& echo "Asia/Shanghai" > /etc/timezone
```
#### 结果
```shell
[root@localhost timezone-alpine]# docker build -t 314315960/alpine-timezone-cst:latest .
Sending build context to Docker daemon  2.048kB
Step 1/2 : FROM alpine:latest
 ---> 055936d39205
Step 2/2 : RUN apk add -U tzdata && rm -f /etc/localtime && ln -sv /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && echo "Asia/Shanghai" > /etc/timezone
 ---> Running in d029b7d53172
fetch http://dl-cdn.alpinelinux.org/alpine/v3.9/main/x86_64/APKINDEX.tar.gz
fetch http://dl-cdn.alpinelinux.org/alpine/v3.9/community/x86_64/APKINDEX.tar.gz
(1/1) Installing tzdata (2019a-r0)
Executing busybox-1.29.3-r10.trigger
OK: 9 MiB in 15 packages
'/etc/localtime' -> '/usr/share/zoneinfo/Asia/Shanghai'
Removing intermediate container d029b7d53172
 ---> 4a4390954bc6
Successfully built 4a4390954bc6
Successfully tagged 314315960/alpine-timezone-cst:latest

```

## 容器启动时处理

我们还可以在容器启动时通过挂载主机时区配置到容器内，前提是主机时区配置文件正常

```shelll
#挂载宿主机/etc/localtime到容器/etc/localtime
[root@localhost timezone-alpine]# docker run -it -v /etc/localtime:/etc/localtime alpine sh
/ # date
Wed Jul  3 17:33:37 CST 2019
#挂载宿主机/usr/share/zoneinfo/Asia/Shanghai到容器/etc/localtime
[root@localhost timezone-alpine]# docker run -it -v /usr/share/zoneinfo/Asia/Shanghai:/etc/localtime alpine sh
/ # date
Wed Jul  3 17:35:01 CST 2019

```

## 进入容器内处理

如果容器删除后重新启动新的容器，还需要我们进入到容器内配置，非常不方便，**个人不建议此方式**。

```shell
[root@localhost timezone-alpine]# docker run -it alpine sh
/ # apk add -U tzdata \
> && rm -f /etc/localtime \
> && ln -sv /usr/share/zoneinfo/Asia/Shanghai /etc/localtime \
> && echo "Asia/Shanghai" > /etc/timezone
fetch http://dl-cdn.alpinelinux.org/alpine/v3.9/main/x86_64/APKINDEX.tar.gz
fetch http://dl-cdn.alpinelinux.org/alpine/v3.9/community/x86_64/APKINDEX.tar.gz
(1/1) Installing tzdata (2019a-r0)
Executing busybox-1.29.3-r10.trigger
OK: 9 MiB in 15 packages
'/etc/localtime' -> '/usr/share/zoneinfo/Asia/Shanghai'
/ # 
```

## kubernetes处理Pod时区不一致

最一劳永逸的方式还是上边Dockerfile中处理，在基础镜像或者服务镜像里面直接配置好。其次我们还可以通过挂载主机时间配置的方式解决

#### 方案一 ：单独Pod中挂载宿主机本地`/etc/localtime`或`/usr/share/zoneinfo/Asia/Shanghai`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: busybox
  namespace: default
spec:
  containers:
  - name: busybox
    image: busybox
    imagePullPolicy: IfNotPresent
    volumeMounts:
    - name: localtime
      mountPath: /etc/localtime
    command: ["sleep", "60"]
  volumes:
  - name: localtime
    hostPath:
      path: /etc/localtime
```

> 如果宿主机 `/etc/localtime` 已存在且时区正确的话，可以直接挂载，如果宿主机本地 `/etc/localtime` 不存在或时区不正确的话，那么可以直接挂载宿主机 `/usr/share/zoneinfo/Asia/Shanghai` 到容器内 `/etc/localtime`，都是可行的

#### 方案二：利用PodPreset给namespace下Pod注入`volumeMounts`

```
apiVersion: settings.k8s.io/v1alpha1
kind: PodPreset
metadata:
  name: allow-tz-env
  namespace: default
spec:
  volumeMounts:
    - mountPath: /etc/localtime
      name: localtime
      readOnly: true
  volumes:
    - name: localtime
      hostPath:
        path: /etc/localtime
```

这样就在`default`这个命名空间内使用了这个allow-tz-env的PodPreset。这样就会使得该空间内所有的对象都挂载了主机的localtime文件。

#### 方案三：PodPreset匹配指定 Pod 加载配置

```yaml
apiVersion: settings.k8s.io/v1alpha1
kind: PodPreset
metadata:
  name: allow-tz-env
  namespace: default
spec:
  selector:
    matchLabels:
      role: busybox-test
  volumeMounts:
    - mountPath: /etc/localtime
      name: localtime
      readOnly: true
  volumes:
    - name: localtime
      hostPath:
        path: /etc/localtime
```

这里 `selector. matchLabels` 通过 Labels 匹配标签包含 `role: busybox-tett` 的 Pod

#### 方案四：PodPreset排除指定Pod加载配置
```
apiVersion: v1
kind: Pod
metadata:
  name: busybox
  namespace: busybox-test
  annotations:
    podpreset.admission.kubernetes.io/exclude: "true"
  labels:
    app: busybox
spec:
  containers:
    - name: busybox
      image: busybox:latest
```

添加了忽略 PodPreset 注入 annotations 后，没有将信息注入进去，时间还是默认的 `UTC` 时间
