---
title: 方便的Docker别名
date: 2019-04-09 03:36:44
tags:
---

> 原文链接: https://hackernoon.com/handy-docker-aliases-4bd85089a3b8

如果您在命令行中使用过Docker，那么您无疑会编写命令快捷方式，或者至少考虑过这样做。

在这篇文章中，我假设您使用docker命令有点舒服，并且可能对潜在有用的别名有点兴趣。如果你是Docker的新手 - 那么这可能不是一个好的开始。

这篇文章是关于更长的，未缩写的命令的节省时间的快捷方式或别名。因为让我们面对它：

“难道没有人有时间输出整个该死的命令！”

快捷方式可以通过多种方式实现。我们在这里考虑的两个是bash shell别名和shell脚本文件。我将共享的任何快捷方式都可以放在脚本文件或别名文件中。每种方法都有其优点和缺点。

这是一个dkps放在shell脚本中的方便（docker ps）快捷方式。如果您选择这种方法，那么您最终会得到一组脚本，您需要将这些脚本复制到全局可访问的路径，例如/usr/shared/bin，使用PATH或类似的东西。您可能还需要系统权限才能使其正常运行。
 ```
#!/bin/sh
docker ps --format '{{.ID}} ~ {{.Names}} ~ {{.Status}} ~ {{.Image}} '
```
另一种方法是将别名命令放在主目录中的.bash_profile文件中。这使得快捷方式可以移植而无需特殊权限
```
#!/bin/sh
alias dkps="docker ps --format '{{.ID}} ~ {{.Names}} ~ {{.Status}} ~ {{.Image}} '"
```
>如果你是bash别名的新手，请查看此[Bash Aliases](https://medium.com/@werbelow/intro-to-bash-aliases-116c4c60afd1)

为了保持简洁，我将导入与docker相关的别名，而不是直接将它们添加到我的.bash_profile或.bash_aliases脚本中。
在我的.bash_profile脚本中，我只需添加以下内容即可加载我的新.docker_aliases文件。
```
if [  -f ~/.docker_aliases ]; then
    . ~/.docker_aliases
fi
```
在我的实际.bash_profile脚本中，我加载了其他别名，例如.git_aliases脚本。
这种方法也非常便携，因为您可以共享您的docker别名。

在介绍我自己的方便的docker别名之前，让我们创建一个容器样本集，我们可以使用它来测试别名。我们将使用[Docker Stack命令](https://docs.docker.com/engine/reference/commandline/stack/)和两个众所周知的服务Redis和MongoDB来实现。不要担心，如果你不熟悉它们，我们实际上不会使用它们。
```
version: "3.4"

networks:
  servicenet:
    driver: overlay
    ipam:
      config:
        -
          subnet: 10.0.9.0/24

services:
  redis:
    image: redis:4.0.8-alpine
    networks:
      - servicenet
    ports:
      - target: 6379
        published: 6379
        protocol: tcp
        mode: ingress
    deploy:
      replicas: 1

  mongo:
    image: mongo:3.4.3
    networks:
      - servicenet
    ports:
      - target: 27017
        published: 27017
        protocol: tcp
        mode: ingress
    deploy:
      replicas: 1
```
启动我们的测试只需要执行docker堆栈部署并传递上面的compose脚本。我们用我们称之为“test”的堆栈名称结束命令。
![image.png](https://upload-images.jianshu.io/upload_images/3481257-409d429729d91a6a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
一旦docker stack加载，我们可以使用docker ps命令验证它
![image.png](https://upload-images.jianshu.io/upload_images/3481257-3c0e894c8c871ab2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
>注意：为简洁起见，我将缩写某些命令的输出。这具有创建更小图像的额外好处。所以请记住，上面的docker ps命令的输出比显示的长得多

我们可以使用docker stack rm命令拆除堆栈。
![image.png](https://upload-images.jianshu.io/upload_images/3481257-95873542051a90a6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

看起来很容易，对吧？ 
在本文的其余部分中显示屏幕截图时，我将使用上面的设置。
现在我们的别名......这样更好是对的吗？ 我们将看到的第一个别名只是为常见的docker命令提供缩写。
我们将看到的第一个别名只是为常见的docker命令提供缩写。
![image.png](https://upload-images.jianshu.io/upload_images/3481257-fc2bacb3e39cc65c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
马上就可以考虑每次使用`dk`替换`docker`和`docker ps`之类的命令时，或者使用`dks`来处理Docker服务命令（如docker service ls）时节省时间。键入dks的节省加起来！
因此，不要键入`docker log`，而是键入：`dkl b7a8` 
使用`dklf`关注日志。
因此，而不是`docker logs -f b7a8`，你可以简单地使用`dklf b7a8`

另一个方便的日志记录别名是`dkln`（docker log by name）命令。
![image.png](https://upload-images.jianshu.io/upload_images/3481257-8857de28326055c6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
它评估反引号（`）中的管道命令，并将结果用作docker logs命令的参数。该grep命令使用第一个参数来过滤docker ps命令的结果。最后，该awk命令将输出的第一个字段作为value参数。

好吧，这可能会令人困惑。让我们仔细看看。

该docker ps命令返回正在运行的容器列表。
![image.png](https://upload-images.jianshu.io/upload_images/3481257-094f7d4b34956268.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
并docker ps | grep redis返回：
![image.png](https://upload-images.jianshu.io/upload_images/3481257-625da9f4413fc186.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
最后，docker ps | grep redis | awk '{print $1}'返回Redis的容器ID：f5f0ed387073
![image.png](https://upload-images.jianshu.io/upload_images/3481257-2e571b9bba4b2ac7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
这允许我们按名称查看任何容器的日志。
![image.png](https://upload-images.jianshu.io/upload_images/3481257-743dcb33027dbe31.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
当然，您需要确保没有多个与名称匹配的容器。如果这样做，那么只需dkl使用带有容器ID 的命令即可。

上面示例中的一个关键点是，您可以通过组合shell命令来构建一些非常强大的别名。当我们查看构建和发布容器的别名时，我们将在本文稍后看到另一个示例。

查看运行容器的状态是构建和测试容器化服务的重要部分。以下别名使我们更容易执行此操作。

在这篇文章的早些时候，我们查看了dkps（Docker PS）命令。
![image.png](https://upload-images.jianshu.io/upload_images/3481257-b87eb0c766ec6641.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
这是命令的作用：
![image.png](https://upload-images.jianshu.io/upload_images/3481257-c5befc84b91c3719.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
另一个有用的别名是`dkstats`（docker stats）命令：
![image.png](https://upload-images.jianshu.io/upload_images/3481257-c61fde1622563883.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
此命令测试是否提供参数，如果是，则应用grep过滤器
![image.png](https://upload-images.jianshu.io/upload_images/3481257-dd3f4203c4a57031.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
这允许我们查看所有容器的统计信息或按特定容器名称过滤。

该dktop命令呈现类似顶部的显示，显示内存，CPU，网络I / O和块I / O.
![](https://upload-images.jianshu.io/upload_images/3481257-760840cb788edb7e.gif?imageMogr2/auto-orient/strip)
实际的别名非常简单：
![image.png](https://upload-images.jianshu.io/upload_images/3481257-0983b8524ad2684c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
您可以使用您更愿意看到的值自由地进行[自定义](https://www.techietown.info/2017/03/docker-container-monitoring-using-docker-stats/)。我选择了一个基本设置，以使显示屏整齐地安装在多窗格iTerm2屏幕中。

在建造集装箱的过程中，有时需要进入集装箱才能环顾四周。这样做通常涉及：

- docker ps查看容器ID列表
- docker exec -it {containerID} / bin / sh
##### 使用我们的别名，这变为：

- dkps
- dke {containerID}
这是一个视频示例：
![](https://upload-images.jianshu.io/upload_images/3481257-57f7ce6ebaae1c23.gif?imageMogr2/auto-orient/strip)

有时需要重新启动服务。该dksb（搬运工服务弹跳）命令允许我们能够做到这一点。

执行此操作的非别名方法需要使用以下docker service scale命令：
```
$ docker service scale test_redis = 0 
$ docker service scale test_redis = 1
```
使用dksb我们只需输入：

$ dksb test_redis 1

![](https://upload-images.jianshu.io/upload_images/3481257-eac18c22bef0b934.gif?imageMogr2/auto-orient/strip)
我们要看的最后一件事是用于构建和发布docker容器的别名。虽然此过程通常涉及两阶段操作，即docker build / docker push，但您可能还需要执行其他任务以进一步自动化该过程。就我而言，我构建了托管NodeJS微服务的docker容器。我的本地构建的一部分涉及检查.dockerignore文件是否存在，然后查看Node package.json文件内部以提取项目的名称和版本。然后使用名称和版本来形成泊坞标签。

这是dkp（docker publish）别名的样子：
![image.png](https://upload-images.jianshu.io/upload_images/3481257-d08af38e1da1fd4f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
上面的别名使用[jq](https://stedolan.github.io/jq/)（命令行JSON处理器）从package.json文件中提取版本信息。

使用`dkp`只需输入节点项目目录并键入，`dkp`然后输入docker hub存储库名称。在下面的示例中，我正在构建和发布[HydraRouter docker](https://github.com/flywheelsports/hydra-router)容器。
![1_P4VYcqkzufYWeeahgjgI_g.gif](https://upload-images.jianshu.io/upload_images/3481257-a5ad790f60f3e0a5.gif?imageMogr2/auto-orient/strip)

我希望你在这篇文章中找到了一些方便的docker别名，或者至少它激发了你使用别名来优化你的命令行工作流程。因为记住 -  没有人有时间 ......

这是完整的  .docker_aliases脚本，包括我们在这篇文章中没有涉及的一些奖励命令。
 ```
#!/bin/sh

alias dm='docker-machine'
alias dmx='docker-machine ssh'
alias dk='docker'
alias dki='docker images'
alias dks='docker service'
alias dkrm='docker rm'
alias dkl='docker logs'
alias dklf='docker logs -f'
alias dkflush='docker rm `docker ps --no-trunc -aq`'
alias dkflush2='docker rmi $(docker images --filter "dangling=true" -q --no-trunc)'
alias dkt='docker stats --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.NetIO}}"'
alias dkps="docker ps --format '{{.ID}} ~ {{.Names}} ~ {{.Status}} ~ {{.Image}}'"

dkln() {
  docker logs -f `docker ps | grep $1 | awk '{print $1}'`
}

dkp() {
  if [ ! -f .dockerignore ]; then
    echo "Warning, .dockerignore file is missing."
    read -p "Proceed anyway?"
  fi

  if [ ! -f package.json ]; then
    echo "Warning, package.json file is missing."
    read -p "Are you in the right directory?"
  fi

  VERSION=`cat package.json | jq .version | sed 's/\"//g'`
  NAME=`cat package.json | jq .name | sed 's/\"//g'`
  LABEL="$1/$NAME:$VERSION"

  docker build --build-arg NPM_TOKEN=${NPM_TOKEN} -t $LABEL .

  read -p "Press enter to publish"
  docker push $LABEL
}

dkpnc() {
  if [ ! -f .dockerignore ]; then
    echo "Warning, .dockerignore file is missing."
    read -p "Proceed anyway?"
  fi

  if [ ! -f package.json ]; then
    echo "Warning, package.json file is missing."
    read -p "Are you in the right directory?"
  fi

  VERSION=`cat package.json | jq .version | sed 's/\"//g'`
  NAME=`cat package.json | jq .name | sed 's/\"//g'`
  LABEL="$1/$NAME:$VERSION"

  docker build --build-arg NPM_TOKEN=${NPM_TOKEN} --no-cache=true -t $LABEL .
  read -p "Press enter to publish"
  docker push $LABEL
}

dkpl() {
  if [ ! -f .dockerignore ]; then
    echo "Warning, .dockerignore file is missing."
    read -p "Proceed anyway?"
  fi

  if [ ! -f package.json ]; then
    echo "Warning, package.json file is missing."
    read -p "Are you in the right directory?"
  fi

  VERSION=`cat package.json | jq .version | sed 's/\"//g'`
  NAME=`cat package.json | jq .name | sed 's/\"//g'`
  LATEST="$1/$NAME:latest"

  docker build --build-arg NPM_TOKEN=${NPM_TOKEN} --no-cache=true -t $LATEST .
  read -p "Press enter to publish"
  docker push $LATEST
}

dkclean() {
  docker rm $(docker ps --all -q -f status=exited)
  docker volume rm $(docker volume ls -qf dangling=true)
}

dkprune() {
  docker system prune -af
}

dktop() {
  docker stats --format "table {{.Container}}\t{{.Name}}\t{{.CPUPerc}}  {{.MemPerc}}\t{{.NetIO}}\t{{.BlockIO}}"
}

dkstats() {
  if [ $# -eq 0 ]
    then docker stats --no-stream;
    else docker stats --no-stream | grep $1;
  fi
}

dke() {
  docker exec -it $1 /bin/sh
}

dkexe() {
  docker exec -it $1 $2
}

dkreboot() {
  osascript -e 'quit app "Docker"'
  countdown 2
  open -a Docker
  echo "Restarting Docker engine"
  countdown 120
}

dkstate() {
  docker inspect $1 | jq .[0].State
}

dksb() {
  docker service scale $1=0
  sleep 2
  docker service scale $1=$2
}

mongo() {
  mongoLabel=`docker ps | grep mongo | awk '{print $NF}'`
  docker exec -it $mongoLabel mongo "$@"
}

redis() {
  redisLabel=`docker ps | grep redis | awk '{print $NF}'`
  docker exec -it $redisLabel redis-cli
}
```

