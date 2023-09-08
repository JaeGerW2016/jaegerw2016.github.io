---
layout: mypost
title: Docker Tutorials 你可能不知道的Docker 命令Tips
categories: [Docker]
---


分享一些在近期学习的简单实用却可能是你不知道的 docker 命令

## Docker events

可以尝试docker events 输出的docker事件给shell脚本中的管道 来做一些script脚本控制

```
#!/bin/bash

docker events -filter 'container=test' |  while read line; do  if [[ ${line} = *"start"* ]];then echo " container started ${line}" ;fi ; done
```

```
#!/bin/bash

docker events --filter 'event=stop' |  while read line; do  if [[ ${line} = *"stop"* ]];then echo " container stopped ${line}" ;fi ; done
```


## Docker remote engine

基本上是一种从本机 `docker cli` 控制远程服务器上的 `docker daemon实例`,使用起来比较简单，我们将使用 `docker context create `选项来创建自定义上下文。
>注意：您的远程docker daemon端上需要 `-H tcp://0.0.0.0:2375` 开放`2375`端口 或者有 ssh 密钥身份验证才能进行操作。否则，就行不通

```
sudo docker context create --docker host=ssh://root@hostname --description 'test remote docker' remote_engine_ssh

docker context use remote_engine_ssh
```

```
sudo docker context create remote_engine_tcp \
--docker host=tcp://remote_engine:2375 \
--description "Connect to remote_engine through tcp"

docker context use remote_engine_tcp
```

虽然把 `docker daemon` 跟 `docker cli` 分开有一定应用场景，但使用這可能會因為已經习惯使用单机架构有一些地方觉得得有些奇怪。其中最明显的就是 `docker run` 跑起來的容器并非在本机上，而是在 `docker daemon`守护进程。这会导致、当你需要把外部的文件拷贝进容器時，docker 會在`docker daemon`守护进程路径下查找文件而非自己`docker cli`所在的路径，比较好的**解决方法就是通过 nfs 或 nas 这类服务，让 `docker daemon`端和`docker cli`端共享文件存储。**


## Configure Docker(container) to use a proxy server

> 此处不是描述如何为 Docker daemon守护进程配置代理

### 配置 Docker 客户端
可以使用 JSON 配置文件为 Docker 客户端添加代理配置，该文件位于`~/.docker/config.json`. 构建和容器使用此文件中指定的配置。

```
{
 "proxies": {
   "default": {
     "httpProxy": "http://proxy.example.com:3128",
     "httpsProxy": "https://proxy.example.com:3129",
     "noProxy": "*.test.example.com,.example.org,127.0.0.0/8"
   }
 }
}
```
运行测试容器验证

```
root@ubuntu:~# docker run --rm alpine sh -c 'env | grep -i  _PROXY'
HTTPS_PROXY=https://proxy.example.com:3129
no_proxy=*.test.example.com,.example.org,127.0.0.0/8
NO_PROXY=*.test.example.com,.example.org,127.0.0.0/8
https_proxy=https://proxy.example.com:3129
http_proxy=http://proxy.example.com:3128
HTTP_PROXY=http://proxy.example.com:3128
```

### 配置每个守护进程的代理设置（不同context设置不同的代理配置） 
`default`下面的键配置proxies客户`daemon.json`端连接到的所有守护程序的代理设置。要为各个守护程序配置代理，请使用守护程序的地址而不是默认在default下面的配置。

以下示例为地址上的 Docker 守护进程配置默认代理配置和无代理覆盖 `tcp://docker-daemon1.example.com`和`ssh://root@hostname`
```
{
 "proxies": {
   "default": {
     "httpProxy": "http://proxy.example.com:3128",
     "httpsProxy": "https://proxy.example.com:3129",
     "noProxy": "*.test.example.com,.example.org,127.0.0.0/8"
   },
   "tcp://docker-daemon1.example.com": {
     "noProxy": "*.internal.example.net"
   },
   "ssh://root@hostname": {
     "noProxy": "*.internal.example.net"
   }
 }
}
```
### 使用 CLI 设置代理
可以在调用`docker build`和`docker run`命令时在命令行上指定代理配置,使用`--build-arg`用于构建的标志，以及`--env`当您想要使用代理运行容器时的标志。

```
$ docker build --build-arg HTTP_PROXY="http://proxy.example.com:3128" .
$ docker run --env HTTP_PROXY="http://proxy.example.com:3128" redis
```

具体文档可以参见[Configure Docker to use a proxy server](https://docs.docker.com/network/proxy/)的内容


## 清理dangling image（悬空镜像）
在日常的`image`镜像构建`build`过程中，总会产生一些忘记打`tag`的镜像，或者在进行分级构建镜像中过程中生成`TAG：<none>`的中间层镜像,这些镜像一般情况下会做定期清理。

**列出 dangling镜像**
```
docker images -f "dangling=true"
```
```
root@ubuntu:~# docker images -f "dangling=true"
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
314315960/hello     <none>              2175c21df3b4        4 months ago        41MB

```
**删除所有的 dangling images**

```
docker rmi $(docker images -f "dangling=true" -q)
```

## 批量操作容器Container
当服务器重启或者因故关机时，而容器的启动参数不带 `--restart always`时，docker 容器可能需要全部重新启动，启动所有 docker 容器

**启动全部容器**

```
docker start $(docker ps -aq)
```

**停止所有 docker 容器**
```
docker stop $(docker ps -aq)
```
**删除全部 docker 容器**

```
docker rm $(docker ps -aq)
```

**删除所有 docker 镜像**
```
docker rmi $(docker images -q)
```

## docker 资源清理

```
docker container prune  # 删除所有退出状态的容器
docker volume prune  # 删除未被使用的数据卷
docker image prune  # 删除 dangling 或所有未被使用的镜像

docker system prune  #删除已停止的容器、dangling 镜像、未被容器引用的 network 和构建过程中的 cache

# 安全起见，这个命令默认不会删除那些未被任何容器引用的数据卷，如果需要同时删除这些数据卷，你需要显式的指定 --volumns 参数

docker system prune --all --force --volumns #这次不仅会删除数据卷，而且连确认的过程都没有了！注意，使用 --all 参数后会删除所有未被引用的镜像而不仅仅是 dangling 镜像
```