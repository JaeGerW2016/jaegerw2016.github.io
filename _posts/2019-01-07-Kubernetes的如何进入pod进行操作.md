---
title: Kubernetes的如何进入pod进行操作
date: 2019-01-07 23:57:54
tags: "kubernetes"
---
首先类似于Docker容器，Kubernetes 使用kubectl 进行命令行操作
### 进入docker容器 ：

```
docker exec -ti  <your-container-name>   /bin/sh
```

### 进入Kubernetes的pod：

```
kubectl exec -ti <your-pod-name>  -n <your-namespace>  -- /bin/sh
```


### [Overview of kubectl](https://kubernetes.io/docs/reference/kubectl/overview/)

`kubectl exec` - **Execute a command against a container in a pod.**
```
// Get output from running 'date' from pod <pod-name>. By default, output is from the first container.
$ kubectl exec <pod-name> date

// Get output from running 'date' in container <container-name> of pod <pod-name>.
$ kubectl exec <pod-name> -c <container-name> date

// Get an interactive TTY and run /bin/bash from pod <pod-name>. By default, output is from the first container.
$ kubectl exec -ti <pod-name> /bin/bash
```
