---
layout:     post
title:     Ephemeral Containers容器入门
subtitle:  Getting Started With Ephemeral Containers
date:       2022-09-20
author:     J
catalog: true
tags:
    - Kubernetes
---
# 背景
如果您正在关注 Kubernetes 生态社区的最新消息，您可能已经听说过 Ephemeral Containers。如果没有？不要害怕！在这篇博文中，尝试阐明这个新功能很快就会在Kubernetes v1.25版本GA

# 什么是Ephemeral Containers
Ephemeral Containers是在Pod已经运行的容器的上下文中运行具有特定镜像的容器。这个对于调试无特定linux发行版的基础容器（例如:distroless）或者缺少某些实用程序的镜像时会派上用场。
对于`kubeclt exec`无法满足调试的时候，`Ephemeral Containers`就有用武之地了。
让我们看一些关于如何使用`Ephemeral Containers`进行调试的示例。

`deployment.yml`
```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: py-serv-deployment
  labels:
    app: py-serv
spec:
  replicas: 1
  selector:
    matchLabels:
      app: py-serv
  template:
    metadata:
      labels:
        app: py-serv
    spec:
      containers:
        - name: py-serv
          image: ghcr.io/metalbear-co/mirrord-pytest:latest
          ports:
            - containerPort: 80
          env:
            - name: MIRRORD_FAKE_VAR_FIRST
              value: mirrord.is.running
            - name: MIRRORD_FAKE_VAR_SECOND
              value: "7777"

---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: py-serv
  name: py-serv
spec:
  ports:
    - port: 80
      protocol: TCP
      targetPort: 80
      nodePort: 30000
  selector:
    app: py-serv
  sessionAffinity: None
  type: NodePort
```
### 部署deployment
```shell
kubectl apply -f deployment.yaml
```
### 列出Pod
```shell
root@node1:~/ephemeral-containers# kubectl get pod
NAME                                 READY   STATUS    RESTARTS   AGE
py-serv-deployment-ff89b5974-8g8dk   1/1     Running   0          81s
```

### 使用Tcpdump工具调试Pod
```shell
root@node1:~/ephemeral-containers# kubectl exec -it py-serv-deployment-ff89b5974-8g8dk -- sh

# tcpdump
sh: 4: tcpdump: not found

```

### 使用ephemeral container容器调试Pod

```shell
root@node1:~/ephemeral-containers# kubectl debug -it py-serv-deployment-ff89b5974-8g8dk --image busybox
Defaulting debug container name to debugger-zljmr.
If you don't see a command prompt, try pressing enter.
/ #

```

### 使用ephemeral container 发送请求

```shell
/ # wget localhost:80
Connecting to localhost:80 (127.0.0.1:80)
saving to 'index.html'
index.html           100% |*****************************************************************************************************************************************|    28  0:00:00 ETA
'index.html' saved
/ # ls
bin         dev         etc         home        index.html  proc        root        sys         tmp         usr         var
/ # cat index.html
OK - GET: Request completed
/ #
```

### 检查网络流量

`nginx-deployment.yml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 1
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.23.1
        ports:
        - containerPort: 80
```
### 部署nginx-deploymnet

```shell
root@node1:~/ephemeral-containers# kubectl apply -f nginx-deployment.yaml
```
### curl 调试nginx Pod
```shell
root@node1:~/ephemeral-containers# kubectl run mycurlpod --image=curlimages/curl -i --tty -- sh
If you don't see a command prompt, try pressing enter.
/ $
/ $ curl http://10.233.68.253  #nginx pod clusterip
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
/ $

```


### ephemeral-containers调试nginx Pod
```shell
root@node1:~/ephemeral-containers# kubectl debug -it nginx-deployment-79bc5ccc66-fv7s4 --image itsthenetwork/alpine-tcpdump -- sh
Defaulting debug container name to debugger-dgp9h.
If you don't see a command prompt, try pressing enter.
/ #
/ # tcpdump -i any port 80
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on any, link-type LINUX_SLL (Linux cooked), capture size 262144 bytes


07:51:54.553852 IP 10.233.66.121.43886 > nginx-deployment-79bc5ccc66-fv7s4.80: Flags [S], seq 633524895, win 64860, options [mss 1410,sackOK,TS val 342237139 ecr 0,nop,wscale 7], length 0
07:51:54.553944 IP nginx-deployment-79bc5ccc66-fv7s4.80 > 10.233.66.121.43886: Flags [S.], seq 2165475887, ack 633524896, win 64308, options [mss 1410,sackOK,TS val 599414041 ecr 342237139,nop,wscale 7], length 0
07:51:54.560898 IP 10.233.66.121.43886 > nginx-deployment-79bc5ccc66-fv7s4.80: Flags [.], ack 1, win 507, options [nop,nop,TS val 342237140 ecr 599414041], length 0
07:51:54.560956 IP 10.233.66.121.43886 > nginx-deployment-79bc5ccc66-fv7s4.80: Flags [P.], seq 1:82, ack 1, win 507, options [nop,nop,TS val 342237140 ecr 599414041], length 81: HTTP: GET / HTTP/1.1
07:51:54.560963 IP nginx-deployment-79bc5ccc66-fv7s4.80 > 10.233.66.121.43886: Flags [.], ack 82, win 502, options [nop,nop,TS val 599414048 ecr 342237140], length 0
07:51:54.561206 IP nginx-deployment-79bc5ccc66-fv7s4.80 > 10.233.66.121.43886: Flags [P.], seq 1:239, ack 82, win 502, options [nop,nop,TS val 599414048 ecr 342237140], length 238: HTTP: HTTP/1.1 200 OK
07:51:54.567245 IP nginx-deployment-79bc5ccc66-fv7s4.80 > 10.233.66.121.43886: Flags [P.], seq 239:854, ack 82, win 502, options [nop,nop,TS val 599414054 ecr 342237140], length 615: HTTP

/ # ping localhost
PING localhost (127.0.0.1): 56 data bytes
64 bytes from 127.0.0.1: seq=0 ttl=64 time=0.037 ms
64 bytes from 127.0.0.1: seq=1 ttl=64 time=0.046 ms
64 bytes from 127.0.0.1: seq=2 ttl=64 time=0.059 ms
64 bytes from 127.0.0.1: seq=3 ttl=64 time=0.048 ms
64 bytes from 127.0.0.1: seq=4 ttl=64 time=0.047 ms
64 bytes from 127.0.0.1: seq=5 ttl=64 time=0.051 ms

--- localhost ping statistics ---
6 packets transmitted, 6 packets received, 0% packet loss
round-trip min/avg/max = 0.037/0.048/0.059 ms

```

## ephemeral-containers是如何工作的
Kubernetes 将给定的镜像安排在与所选容器相同的命名空间中运行。但是让我们仔细看看哪些命名空间可用于临时容器。

`kubectl exec`调试的 pod，列出所有命名空间，并将它们与ephemeral-containers中设置的命名空间进行比较。

### py-serv的命名空间

```shell
root@node1:~# kubectl exec -it py-serv-deployment-ff89b5974-8g8dk -- sh
Defaulted container "py-serv" out of: py-serv, debugger-zljmr (ephem), debugger-6jbrv (ephem)
# ls -l /proc/self/ns
total 0
lrwxrwxrwx 1 root root 0 Sep 20 08:11 cgroup -> 'cgroup:[4026532738]'
lrwxrwxrwx 1 root root 0 Sep 20 08:11 ipc -> 'ipc:[4026532730]'
lrwxrwxrwx 1 root root 0 Sep 20 08:11 mnt -> 'mnt:[4026532736]'
lrwxrwxrwx 1 root root 0 Sep 20 08:11 net -> 'net:[4026532625]'
lrwxrwxrwx 1 root root 0 Sep 20 08:11 pid -> 'pid:[4026532737]'
lrwxrwxrwx 1 root root 0 Sep 20 08:11 pid_for_children -> 'pid:[4026532737]'
lrwxrwxrwx 1 root root 0 Sep 20 08:11 time -> 'time:[4026531834]'
lrwxrwxrwx 1 root root 0 Sep 20 08:11 time_for_children -> 'time:[4026531834]'
lrwxrwxrwx 1 root root 0 Sep 20 08:11 user -> 'user:[4026531837]'
lrwxrwxrwx 1 root root 0 Sep 20 08:11 uts -> 'uts:[4026532729]'

```

### ephem的命名空间

```shell
root@node1:~/ephemeral-containers# kubectl debug -it py-serv-deployment-ff89b5974-8g8dk --image busybox -- sh
Defaulting debug container name to debugger-6jbrv.
If you don't see a command prompt, try pressing enter.
/ #
/ # ls -l /proc/self/ns
total 0
lrwxrwxrwx    1 root     root             0 Sep 20 08:07 cgroup -> cgroup:[4026532846]
lrwxrwxrwx    1 root     root             0 Sep 20 08:07 ipc -> ipc:[4026532730]
lrwxrwxrwx    1 root     root             0 Sep 20 08:07 mnt -> mnt:[4026532844]
lrwxrwxrwx    1 root     root             0 Sep 20 08:07 net -> net:[4026532625]
lrwxrwxrwx    1 root     root             0 Sep 20 08:07 pid -> pid:[4026532845]
lrwxrwxrwx    1 root     root             0 Sep 20 08:07 pid_for_children -> pid:[4026532845]
lrwxrwxrwx    1 root     root             0 Sep 20 08:07 time -> time:[4026531834]
lrwxrwxrwx    1 root     root             0 Sep 20 08:07 time_for_children -> time:[4026531834]
lrwxrwxrwx    1 root     root             0 Sep 20 08:07 user -> user:[4026531837]
lrwxrwxrwx    1 root     root             0 Sep 20 08:07 uts -> uts:[4026532729]
```

看起来ephemeral-container具有相同的ipc、net、user、time和uts命名空间。mnt命名空间不可用是有意义的，因为二者文件系统是不同的。如果明确指定了目标容器，则临时容器可以访问 pid 命名空间。这意味着我们可以通过引用根路径来访问被调试 pod 的文件系统`/proc/1/root`

```shell
root@node1:~/ephemeral-containers# kubectl debug -it py-serv-deployment-ff89b5974-8g8dk --image busybox -- sh
Defaulting debug container name to debugger-6jbrv.
If you don't see a command prompt, try pressing enter.
/ #
/ # ps
PID   USER     TIME  COMMAND
    1 root      0:00 sh
    9 root      0:00 ps
/ # cd /proc/1/root
/ # ls
bin   dev   etc   home  proc  root  sys   tmp   usr   var
```

# 结论
ephemeral-container容器确实非常有用，被调试容器的基础镜像不需要包含特定工具或者基于Linux发行版构建，因为它们省去了切换命名空间和与各种容器运行时交互的麻烦。虽然 Kubernetes 某些操作需要特权运行，但我们可以在使用ephemeral-container容器时轻松摆脱特权安全上下文，因为我们不需要挂载容器运行时套接字。
