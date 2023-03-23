---
title:  Azure中国提供了gcr.io/k8s.gcr.io/quay.io镜像代理服务
date: 2019-07-22 10:13
tag: kubernetes
---

Azure China提供了目前用过的质量最好的镜像源。无论是速度还是覆盖范围。而且都支持匿名拉取，也就是不需要登录,体验非常友好。

拉取时需要改一下前缀，等pull完了以后再tag为新的镜像名。

```shell
#gcr.io
docker pull gcr.azk8s.cn/google_containers/<imagename>:<version>

#quay.io
docker pull quay.azk8s.cn/<repo-name>/<image-name>:<version>

#docker.io
docker pull dockerhub.azk8s.cn/<repo-name>/<image-name>:<version>
```

> Note: `k8s.gcr.io` would redirect to `gcr.io/google_containers`, following image urls are identical:

```shell
k8s.gcr.io/pause-amd64:3.1
gcr.io/google_containers/pause-amd64:3.1
```

[Docker-wrapper](https://github.com/JaeGerW2016/containerTools/tree/master/docker-wrapper)

Docker-wrapper 这个脚本可以自动根据镜像名称进行解析，转换为azure的mirror镜像源域名。并进行拉取。拉取完成后会自动进行tag重命名为原本的镜像名

### install

```shell
git clone git@github.com:JaeGerW2016/containerTools.git
 
cp containerTools/docker-wrapper/docker-wrapper /usr/local/bin/
```

### Usage

```shell
[root@localhost ~]# docker-wrapper pull gcr.io/google_containers/hyperkube-amd64:v1.13.5
-- pull gcr.io/google_containers/hyperkube-amd64:v1.13.5 from gcr.azk8s.cn instead --
v1.13.5: Pulling from google_containers/hyperkube-amd64
73e3e9d78c61: Pull complete 
26fd5cc874d4: Pull complete 
71154a383818: Pull complete 
dc72cda1a886: Pull complete 
fb2d9bd8c41e: Pull complete 
16c0e1b4fd78: Pull complete 
6987d95b1d79: Pull complete 
904ad0f3a448: Pull complete 
Digest: sha256:548b8fa78eed795509184885ad6ddc6efc1e8cae3b21729982447ca124dfe364
Status: Downloaded newer image for gcr.azk8s.cn/google_containers/hyperkube-amd64:v1.13.5
Untagged: gcr.azk8s.cn/google_containers/hyperkube-amd64:v1.13.5
Untagged: gcr.azk8s.cn/google_containers/hyperkube-amd64@sha256:548b8fa78eed795509184885ad6ddc6efc1e8cae3b21729982447ca124dfe364
-- pull gcr.io/google_containers/hyperkube-amd64:v1.13.5 done --

[root@localhost ~]# docker-wrapper pull quay.io/coreos/prometheus-operator:v0.29.0
-- pull quay.io/coreos/prometheus-operator:v0.29.0 from quay.azk8s.cn instead --
v0.29.0: Pulling from coreos/prometheus-operator
57c14dd66db0: Pull complete 
65b1a901dccc: Pull complete 
1fc21e379f9a: Pull complete 
Digest: sha256:5abe9bdfd93ac22954e3281315637d9721d66539134e1c7ed4e97f13819e62f7
Status: Downloaded newer image for quay.azk8s.cn/coreos/prometheus-operator:v0.29.0
Untagged: quay.azk8s.cn/coreos/prometheus-operator:v0.29.0
Untagged: quay.azk8s.cn/coreos/prometheus-operator@sha256:5abe9bdfd93ac22954e3281315637d9721d66539134e1c7ed4e97f13819e62f7
-- pull quay.io/coreos/prometheus-operator:v0.29.0 done --

[root@localhost ~]# docker-wrapper pull nginx
Using default tag: latest
latest: Pulling from library/nginx
0a4690c5d889: Pull complete 
9719afee3eb7: Pull complete 
44446b456159: Pull complete 
Digest: sha256:b4b9b3eee194703fc2fa8afa5b7510c77ae70cfba567af1376a573a967c03dbb
Status: Downloaded newer image for nginx:latest

```

### 参考文档：
http://mirror.azure.cn/help/gcr-proxy-cache.html
https://github.com/Azure/container-service-for-azure-china/tree/master/aks#22-container-registry-proxy
