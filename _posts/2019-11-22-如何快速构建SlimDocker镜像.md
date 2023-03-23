---
title: 如何快速构建Slim-Docker镜像
date: 2019-11-22 11:23
tag: docker
---

众所周知，Docker是容器化应用的主要参与者，并且Docker镜像无处不在。这很棒，因为您可以轻松地并排启动例如不同版本的数据库。将您的应用程序图像合并在一起也非常简单。这是由于大量的基础镜像和简单的定义语言所致。但是，当您在不知道自己在做什么的情况下简单的构建一个应用镜像的时候，就会遇到以下*两个问题*。

1. **由于镜像安装没必要的组件，您浪费了磁盘空间。**
2. **镜像等待构建时间过长会浪费时间。**

在本文中，我想向您展示如何解决这两个问题。幸运的是，这仅需要您了解Docker提供的一些技巧和技术。为了使本教程有趣且有用，我向您展示了如何将Python App打包到Docker映像中。您可以在我的[**Github repository**](https://github.com/JaeGerW2016/docker-image-slim-tutorial)找到下面引用的所有代码。

### 教程

这里我们简单构建了一个python文件main.py，们的应用程序仅仅是一个简单的网络服务器，它依赖于`pandas`，`fastapi`和`uvicorn`。我们将依赖项写在`requirements.txt`文件中

`main.py`

```
from fastapi import FastAPI

app = FastAPI()


@app.get("/")
def read_root():
    return {"Hello": "World"}


@app.get("/items/{item_id}")
def read_item(item_id: int, q: str = None):
    return {"item_id": item_id, "q": q}
```

`requirements.txt`

```
pandas==0.25.3
fastapi
uvicorn==0.10.8
```

**未优化前的`Dockerfile`**

```Dockerfile
FROM python:3.8.0-slim
COPY . /app
RUN apt-get update \
&& apt-get install gcc -y \
&& apt-get clean
WORKDIR app
RUN pip install --user --default-timeout=100 -r requirements.txt -i https://pypi.doubanio.com/simple/
ENTRYPOINT uvicorn main:app --reload --host 0.0.0.0 --port 8080
```

```shell
root@ubuntu:~/tutorial# ls
Dockerfile  main.py  requirements.txt
root@ubuntu:~/tutorial# docker build -t my-app:v1 .

root@ubuntu:~/tutorial# docker images | grep my-app
REPOSITORY                                          TAG                 IMAGE ID            CREATED             SIZE
my-app                                              v1                  a65501a661da        39 minutes ago      529MB
```

这个my-app的镜像大小为529MB,大概使用2分钟时间以上 

> 基础镜像用`python:3.8.0-slim`，考虑到完整的Ubuntu和Centos镜像大小在1G以上，但是我们这个python app只需要python环境，无需安装其他组件，考虑`python:3.8.0-alpine`的镜像大小更小，但是由于python app 依赖pandas，这个在alpine基础镜像上安装很麻烦，而且alpine在安全性和稳定性上也有一定的问题，所以基于以上的综合考虑，采用`python:3.8.0-slim`，而且`slim`版的镜像大小只比`alpine`的大80MB

### 忽略构建上下文

docker 当我们在 `docker build` 的过程中，首先会将指定的上下文目录打包传递给 `docker引擎`，忽略这部分构建的上下文，从而减小镜像的大小

`.dockerignore`

```
# Ignore .git folder
.git*
# Ignore all Windows, Linux and OSX special files
# https://www.gitignore.io/api/windows,linux,osx,vim
# Ignore all your language-specific build folders
build/
target/
node_modules/
.bundler/
etc
venv
.venv
```

### 层缓存

有些时候如果需要频繁构建镜像，那这个时候层缓存就非常的有必要，可以大大节约构建的时间。Dockerfile中每一行都代表一层，通过添加/删除行中的内容，或者更改其引用的文件或文件夹，可以改变镜像层，发生这种情况之后，当前镜像层和其下面的镜像层都将被重新构建。如果镜像层没有变更，则可以利用缓存进行快速构建。

因此要利用好镜像层缓存，就需要对Dockerfile进行结构化

- 对于频繁构建过程中经常不变的层应该出现在Dockerfile的开头部分，这里安装`gcc`编译器就是一个例子。
- 对于频繁构建过程中经常变化的层应该出现在Dockerfile的结尾部分，COPY变化的代码就是一个例子。

**进行一定优化后的`Dockerfile`**

```
FROM python:3.8.0-slim
RUN apt-get update \
&& apt-get install gcc -y \
&& apt-get clean
COPY requirements.txt /app/requirements.txt
WORKDIR app
RUN pip install --user --default-timeout=100 -r requirements.txt -i https://pypi.doubanio.com/simple/
COPY . /app
ENTRYPOINT uvicorn main:app --reload --host 0.0.0.0 --port 1234
```

以上的dockerfile变化的并不大，对于安装`gcc`和python 依赖 这部分不太变化的部分写在前面，将COPY 操作写在后面，这样就能最大程度的利用层缓存，不会重新安装依赖项，大大缩短了构建时间

```
root@ubuntu:~/tutorial# docker images
REPOSITORY                                          TAG                 IMAGE ID            CREATED             SIZE
my-app                                              v2                  fe959ec48872        8 seconds ago       529MB
```

### 多阶段构建

我们必须安装GCC才能安装FastApi和uvicorn。但是， 对于**运行该应用程序，我们不需要编译器**。现在想象您不仅需要GCC，还需要其他程序，例如Git，CMake，NPM或…。您的生产镜像变得越来越臃肿。

> 这个时候多阶段构建，可以助我们一臂之力

通过多阶段构建，您可以在同一Dockerfile中定义各种镜像。每个镜像执行不同的步骤。您可以将从一个镜像生成的文件和工件复制到另一个镜像。最常见的情况是，您有一个用于构建应用程序的镜像，另一个用于运行它。您需要做的就是将构建基础镜像和依赖项从构建映像复制到应用程序映像。

```dockerfile
# Here is the build image
FROM python:3.8.0-slim as builder
RUN apt-get update \
&& apt-get install gcc -y \
&& apt-get clean
COPY requirements.txt /app/requirements.txt
WORKDIR app
RUN pip install --user --default-timeout=100 -r requirements.txt -i https://pypi.doubanio.com/simple/
COPY . /app
# Here is the production image
FROM python:3.8.0-slim as app
COPY --from=builder /root/.local /root/.local
COPY --from=builder /app/main.py /app/main.py
WORKDIR app
ENV PATH=/root/.local/bin:$PATH
ENTRYPOINT uvicorn main:app --reload --host 0.0.0.0 --port 1234
```

```
root@ubuntu:~/tutorial# docker build -t my-app:v3 .
root@ubuntu:~/tutorial# docker images
REPOSITORY                                          TAG                 IMAGE ID            CREATED             SIZE
my-app                                              v3                  677fed4b1cec        11 seconds ago      353MB
```

利用多阶段构建将用于构建应用程序的镜像和运行应用程序的镜像分开之后，镜像大小精简到353MB



### 总结

在本文中，我向您展示了一些简单的技巧和窍门，说明了如何创建更快构建的较小的Docker映像。记得

- **始终添加.dockerignore文件。**
- **考虑一下镜像构建命令顺序，并将不变的镜像层放在前面，频繁变更的镜像层放在最后。**
- **尝试使用和利用多阶段构建。**

我希望这会在将来节省一些磁盘空间和时间
