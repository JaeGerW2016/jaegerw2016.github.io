---
layout: mypost
title: 如何使用Docker Buildx构建多平台架构镜像
categories: [Container，Docker]
---
## 背景
Docker作为容器管理和应用容器化的重要工具，随着现在应用容器化的程度越来越高，我们在享受其提供的便捷的方式来打包和部署应用程序。但是Apple的M1和M2芯片的在基于Arm架构发展和亚马逊也将发展基于 AWS Graviton2 Arm 架构处理器，对于客户在 AWS 上构建基于 Arm 的原生应用程序，拥有了更加丰富的选择，同时为在 Amazon EC2 中运行的工作负载提供了出色的性价比。

这样带来的问题就是之前在Intel x86架构下构建的应用程序无法运行在Arm架构处理器的实例上面，会显示警告并且容器突然终止。

>WARNING: The requested image's platform (linux/arm64) does not match the detected host platform (linux/amd64) and no specific platform was requested

经过一些研究，我发现了一个叫做 `docker buildx`构建方式

默认的` docker build `命令无法完成跨平台构建任务，我们需要为 `docker `CLI命令行安装 **buildx 插件**扩展其功能。buildx 能够使用由 Moby BuildKit 提供的构建镜像额外特性，它能够创建多个 builder 实例，在多个节点并行地执行构建任务，以及跨平台构建。

## Docker buildx
如果你的 docker 没有 buildx 命令，可以下载二进制包进行安装：

方式一

首先从 Docker buildx 项目的 release 页面找到适合自己平台的二进制文件。
下载二进制文件到本地并重命名为 docker-buildx，移动到 docker 的插件目录 ~/.docker/cli-plugins
向二进制文件授予可执行权限。
```
 $ mkdir -p ~/.docker/cli-plugins
 $ cd ~/.docker/cli-plugins
 $ wget https://github.com/docker/buildx/releases/download/v0.11.2/buildx-v0.11.2.linux-amd64
 $ mv buildx-v0.11.2.linux-amd64 docker-buildx
 $ chmod +x docker-buildx
```

方式二

如果本地的 docker 版本高于 19.03，可以通过以下命令直接在本地构建并安装，这种方式更为方便：
```
$ DOCKER_BUILDKIT=1 docker build --platform=local -o . "https://github.com/docker/buildx.git"
$ mkdir -p ~/.docker/cli-plugins
$ mv buildx ~/.docker/cli-plugins/docker-buildx
```
使用 buildx 进行构建的方法如下：
```
docker buildx build .
```
`docker buildx` 和 `docker build` 命令的使用体验基本一致，还支持 build 常用的选项如 `-t -f `等。


## builder实例
docker buildx 通过 builder 实例对象来管理构建配置和节点，命令行将构建任务发送至 builder 实例，再由 builder 指派给符合条件的节点执行。我们可以基于同一个 docker 服务程序创建多个 builder 实例，提供给不同的项目使用以隔离各个项目的配置，也可以为一组远程 docker 节点创建一个 builder 实例组成构建阵列，并在不同阵列之间快速切换。

使用 docker buildx create 命令可以创建 builder 实例，这将以当前使用的 docker 服务为节点创建一个新的 builder 实例。要使用一个远程节点，可以在创建示例时通过 DOCKER_HOST 环境变量指定远程端口或提前切换到远程节点的 docker context。

当前节点创建一个新的 builder 实例，并通过命令行选项指定实例名称、驱动以及当前节点的目标平台：

```
root@debian-wmctmfwjmq ~ # docker buildx create --name mybuilder
mybuilder
root@debian-wmctmfwjmq ~ # docker buildx ls
NAME/NODE    DRIVER/ENDPOINT             STATUS  BUILDKIT PLATFORMS
mybuilder *  docker-container                             
  mybuilder0 unix:///var/run/docker.sock running v0.11.6  linux/amd64, linux/amd64/v2, linux/amd64/v3, linux/386
default      docker                                       
  default    default                     running 23.0.4   linux/amd64, linux/amd64/v2, linux/amd64/v3, linux/386
```
```
root@debian-wmctmfwjmq ~ # docker ps
CONTAINER ID   IMAGE                           COMMAND       CREATED        STATUS        PORTS     NAMES
0d1b9d05413a   moby/buildkit:buildx-stable-1   "buildkitd"   20 hours ago   Up 20 hours             buildx_buildkit_mybuilder0

```
如下将把一个远程节点加入 builder 实例，这里`--append` 追加远程节点,这里需要能访问`DOCKER_HOST` 这个地址，不然在创建的时候会报错
```
$ export DOCKER_HOST=tcp://10.100.xx.yy:2375
$ docker buildx create --name mybuilder --append --node remote-builder
```
如上就创建了一个支持多平台架构的 builder 实例，

执行 `docker buildx use <builder>` 将切换到所指定的 builder 实例。

以下3个命令用于管理一个builder 实例的生命周期。

- `docker buildx inspect`
- `docker buildx stop`
- `docker buildx rm `

## 跨平台构建

`docker buildx build `通过 `--platform` 选项指定构建的目标平台。Dockerfile 中的 FROM 指令如果没有设置 `--platform` 标志，就会以目标平台拉取基础镜像，最终生成的镜像也将属于目标平台。此外 Dockerfile 中可通过 `BUILDPLATFORM、TARGETPLATFORM、BUILDARCH 和 TARGETARCH `等参数使用该选项的值。当使用 docker-container 驱动时，这个选项可以接受用逗号分隔的多个值作为输入以同时指定多个目标平台，所有平台的构建结果将合并为一个整体的镜像列表作为输出，因此无法直接输出为本地的 docker images 镜像。

`docker buildx build `支持丰富的输出行为，通过`--output=[PATH,-,type=TYPE[,KEY=VALUE]` 选项可以指定构建结果的输出类型和路径等，常用的输出类型有以下几种：

`local`：构建结果将以文件系统格式写入 dest 指定的本地路径， 如 --output type=local,dest=./output。
`tar`：构建结果将在打包后写入 dest 指定的本地路径。
`oci`：构建结果以 OCI 标准镜像格式写入 dest 指定的本地路径。
`docker`：构建结果以 Docker 标准镜像格式写入 dest 指定的本地路径或加载到 docker 的镜像库中。同时指定多个目标平台时无法使用该选项。
`image`：以镜像或者镜像列表输出，并支持 push=true 选项直接推送到远程仓库，同时指定多个目标平台时可使用该选项。
`registry`：type=image,push=true 的精简表示。

对本示例我们执行如下 docker buildx build 命令：

```shell
$ docker buildx build --platform linux/amd64,linux/arm64 -t 314315960/redis-trib -o type=registry .

#以下--push 和 -o type=registry 效果一样，直接推送镜像到远程镜像仓库
$ docker buildx build --platform linux/amd64,linux/arm64 -t 314315960/redis-trib --push .

```
该命令将在当前目录同时构建 linux/amd64、 linux/arm64  二种平台的镜像，并将输出结果直接推送到远程的DockerHub镜像仓库中。
构建过程可拆解如下：

- docker 将构建上下文传输给 builder 实例。
- builder 为命令行 --platform 选项指定的每一个目标平台构建镜像，包括拉取基础镜像和执行构建步骤。
- 导出构建结果，镜像文件层被推送到远程仓库。
- 生成一个清单 JSON 文件，并将其作为镜像标签推送给远程仓库。

## 验证构建结果

```shell
root@debian-wmctmfwjmq ~/muti-arch-image # docker buildx imagetools inspect 314315960/redis-trib:latest
Name:      docker.io/314315960/redis-trib:latest
MediaType: application/vnd.oci.image.index.v1+json
Digest:    sha256:dd10c5a122b48ecf329909152cf066ecf8665eb53446416e66f3f1e09718a1c0

Manifests:
  Name:        docker.io/314315960/redis-trib:latest@sha256:74671e4350b6675a91db25e539a544d9ee5168886cf6dcd3c1211aee719bcbc1
  MediaType:   application/vnd.oci.image.manifest.v1+json
  Platform:    linux/amd64

  Name:        docker.io/314315960/redis-trib:latest@sha256:a53012f48776900963d386e259644f91f722e9a22a715b4be87169a96bf3dcdb
  MediaType:   application/vnd.oci.image.manifest.v1+json
  Platform:    linux/arm64

```

## 如何交叉编译 Golang 的 CGO 项目 [转]
支持交叉编译到常见的操作系统和 CPU 架构是 Golang 的一大优势，但以上示例中的解决方案无需修改Dockerfile的普通代码，如果项目中通过 cgo 调用了 C 代码，情况会变得更加复杂。

为了能够顺利编译 C 代码到目标平台，首先需要在编译环境中安装目标平台的 C 交叉编译器（通常基于 gcc），常用的 Linux 发行版会提供大部分平台的交叉编译器安装包，可以直接通过包管理器安装。

其次还需要安装目标平台的 C 标准库（通常标准库会作为交叉编译器的安装依赖，不需要单独安装），另外取决于你所调用的 C 代码的依赖关系，可能还需要安装一些额外的 C 依赖库（如 libopus-dev 之类）。

我们将使用 amd64 架构的 golang:1.14 官方镜像作为基础镜像执行编译，其使用的 Linux 发行版为 Debian。假设交叉编译的目标平台是 linux/arm64，则需要准备的交叉编译器为 gcc-aarch64-linux-gnu，C 标准库为 libc6-dev-arm64-cross，安装方式为：

$ apt-get update
$ apt-get install gcc-aarch64-linux-gnu
libc6-dev-arm64-cross 会同时被安装。

得益于 Debian 包管理器 dpkg 提供的多架构安装能力，假如我们的代码依赖 libopus-dev 等非标准库，可通过 <library>:<architecture> 的方式安装其 arm64 架构的安装包：

$ dpkg --add-architecture arm64
$ apt-get update
$ apt-get install -y libopus-dev:arm64
交叉编译 CGO 示例
假设有如下 cgo 的示例代码：
```golang
package main

/*
#include <stdlib.h>
*/
import "C"
import "fmt"

func Random() int {
    return int(C.random())
}

func Seed(i int) {
    C.srandom(C.uint(i))
}

func main()  {
    rand := Random()
    fmt.Printf("Hello %d\n", rand)
}
```
将使用的 Dockerfile 如下：
```
FROM --platform=$BUILDPLATFORM golang:1.14 as builder

ARG TARGETARCH
RUN apt-get update && apt-get install -y gcc-aarch64-linux-gnu

WORKDIR /app
COPY . /app/

RUN if [ "$TARGETARCH" = "arm64" ]; then CC=aarch64-linux-gnu-gcc && CC_FOR_TARGET=gcc-aarch64-linux-gnu; fi && \
  CGO_ENABLED=1 GOOS=linux GOARCH=$TARGETARCH CC=$CC CC_FOR_TARGET=$CC_FOR_TARGET go build -a -ldflags '-extldflags "-static"' -o /main main.go
```
Dockerfile 中通过 apt-get 安装了` gcc-aarch64-linux-gnu `作为交叉编译器，示例程序较为简单因此不需要额外的依赖库。在执行 go build 进行编译时，需要通过 `CC` 和 `CC_FOR_TARGET` 环境变量指定所使用的交叉编译器。

为了基于同一份 Dockerfile 执行多个目标平台的编译（假设目标架构只有 amd64/arm64），最下方的 RUN 指令使用了一个小技巧，通过 Bash 的条件判断语法来执行不同的编译命令：

假如构建任务的目标平台是 arm64，则指定 CC 和 CC_FOR_TARGET 环境变量为已安装的交叉编译器（注意它们的值有所不同）。
假如构建任务的目标平台是 amd64，则不指定交叉编译器相关的变量，此时将使用默认的 gcc 作为编译器。
最后使用 buildx 执行构建的命令如下：

```
$ docker buildx build --platform linux/amd64,linux/arm64 -t registry.cn-hangzhou.aliyuncs.com/waynerv/cgo-demo -o type=registry .
```
## 总结

有了 `Buildx 插件`的帮助，在缺少基础设施的情况下，我们也能使用 docker 方便地构建跨平台的应用镜像。

但默认通过 QEMU 虚拟化目标平台指令的方式有明显地性能瓶颈，如果编写应用的语言支持交叉编译，我们可以通过结合 buildx 和交叉编译获得更高的效率。

## 参考文档
[building-multi-architecture-images-with-docker-buildx](https://waynerv.com/posts/building-multi-architecture-images-with-docker-buildx/)
[Multi-platform images](https://docs.docker.com/build/building/multi-platform/)
[buildx github page](https://github.com/docker/buildx)
[using-docker-buildx-to-create-cross-platform-docker-images-for-seamless-compatibility-4k8b](https://dev.to/aws-builders/using-docker-buildx-to-create-cross-platform-docker-images-for-seamless-compatibility-4k8b)
