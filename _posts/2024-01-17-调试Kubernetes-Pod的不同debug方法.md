---
layout: mypost
title:  kubernetes教程：如何通过namespace cgroup和strace来调试pod
categories: [kubernetes,namespace,cgroup,strace]
---

### 背景

`kubernetes`因其复杂的网络环境和容器运行时的安全隔离机制，使得在 `Kubernetes` 的`Pod`内调试运行的应用程序可能是一项具有挑战性的任务。

特别是在构建容器镜像的时候是在基于安全考虑的的基础镜像，例如：`scratch`以及`RedHat` 的通用基础镜像`（UBI）`和谷歌的`distroless`镜像，这些镜像缺乏基本的调试工具时。当应用程序部署在这些基础镜像之上时，无法通过**`kubectl exec  `**进到Pod内部调用 `shell`进行调试 。

对于以上的情况，一般情况下，可以通过修改 `Dockerfile`里的基础镜像，镜像中包含必要的二进制文件，或者在主应用程序旁边部署 `sidecar` 容器或者`debug`容器。但是，这些不适合生产工作负载。



基于以上的情况，考虑容器化利用的**Linux** 的`cgroup`和`Namespace`的强大的资源隔离方式，从而实现高效、安全的应用程序执行。所以我们这次的调试的方式也是利用这个原理来实现的。

**在本文中，将演示如何通过检查`Namespace` `Cgroup` 以及`starce`来调试正在运行的 `Kubernetes Pod` 的分步指南。**



### 准备条件

- **Kubernetes Cluster**
- **Kubernetes Node 的访问权限**
- **kubectl**

可以通过`minikube`起一个本地`Kubernetes cluster` 这里就不做详细的说明，可以参考 [使用 Minikube 创建 Kubernetes 集群](https://jaegerw2016.github.io/posts/2022/11/10/Creating-Kubernetes-Cluster-With-Minikube.html)



本次利用之前升级的的`kubernetes v1.28.3`集群 

```shell
root@node1:~# kubectl get node -owide
NAME    STATUS   ROLES           AGE   VERSION   INTERNAL-IP     EXTERNAL-IP   OS-IMAGE                         KERNEL-VERSION   CONTAINER-RUNTIME
node1   Ready    control-plane   71d   v1.28.3   192.168.2.220   <none>        Debian GNU/Linux 12 (bookworm)   6.1.0-13-amd64   containerd://1.7.7
node2   Ready    <none>          71d   v1.28.3   192.168.2.243   <none>        Debian GNU/Linux 12 (bookworm)   6.1.0-13-amd64   containerd://1.7.7
node3   Ready    <none>          71d   v1.28.3   192.168.2.222   <none>        Debian GNU/Linux 12 (bookworm)   6.1.0-13-amd64   containerd://1.7.7

```



#### 部署示例程序

`faultapp-deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: faultyapp
  labels:
    app.kubernetes.io/name: faultyapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: faultyapp
  template:
    metadata:
      labels:
        app.kubernetes.io/name: faultyapp
    spec:
      securityContext:
        runAsUser: 1000
        runAsGroup: 3000
        fsGroup: 2000     
      containers:
        - name: faultyapp
          imagePullPolicy: Always
          image: cloudziu/faultyapp:latest
          securityContext:
            allowPrivilegeEscalation: false
            privileged: false        
            capabilities:
              drop:
                - all           
          resources:
            requests:
              cpu: 50m
              memory: 100Mi
            limits:
              cpu: 100m
              memory: 100Mi
```

```shell

root@node1:~# kubectl apply -f https://raw.githubusercontent.com/cloudziu/debugging-scratch/master/k8s-deployment.yaml
root@node1:~# kubectl get pod faultyapp-56bbb4bff6-8qjr2 
NAME                         READY   STATUS    RESTARTS   AGE
faultyapp-56bbb4bff6-8qjr2   1/1     Running   0          20h

```

可以看到目前 `falutapp`已经在集群里的状态为`Running` 接下来我们可以**`kubectl logs`** 查看该程序在后台的**`log`**



```shell
root@node1:~# kubectl logs faultyapp-56bbb4bff6-8qjr2  | head -n 5
Starting HTTP ...
Something went wrong...
Something went wrong...
Something went wrong...
Something went wrong...

```

以上的日志，对于应用程序的调试并没有提供太多的信息，从log来看只是提示有报错，这是需要我们**`kubectl exec`** 进入到Pod内部查看



```shell
root@node1:~# kubectl exec -it faultyapp-56bbb4bff6-8qjr2 -- /bin/bash
error: Internal error occurred: error executing command in container: failed to exec in container: failed to start exec "b278e1c619f29f8020bfe8996cb7431626c1186922f043d403d8b3b62f2e4f5c": OCI runtime exec failed: exec failed: unable to start container process: exec: "/bin/bash": stat /bin/bash: no such file or directory: unknown
root@node1:~# kubectl exec -it faultyapp-56bbb4bff6-8qjr2 -- /bin/sh
error: Internal error occurred: error executing command in container: failed to exec in container: failed to start exec "8c1ced9606faed07ef532c51fbb32201ba713963dd0ed471ecaf23c12cf38a29": OCI runtime exec failed: exec failed: unable to start container process: exec: "/bin/sh": stat /bin/sh: no such file or directory: unknown

```

以上信息 可以看出`falutapp`是基于`scratch`基础镜像构建的，未包含`/bin/sh`和`/bin/bash` 二进制文件或者文件目录导致的报错。



### 访问Kubernetes Node 调试过程



通过查看该`falutapp` ` `Pod`是被调度到`node1`节点上，所以接下来的所有操作都是在 `node1`上对其进行`namespace` `cgroup v2` `strace`等一系列操作。



```shell
root@node1:~# PODS_UID=`kubectl get pod faultyapp-56bbb4bff6-8qjr2 -o jsonpath='{.metadata.uid}' | sed 's/-/_/g'`
root@node1:~# echo $PODS_UID
9ea28e70_aa5b_49a2_81ce_c95ae0f6f5a5
root@node1:~# PODS_QOS=`kubectl get pod faultyapp-56bbb4bff6-8qjr2 -o jsonpath='{.status.qosClass}' | tr '[:upper:]' '[:lower:]'`
root@node1:~# echo $PODS_QOS
burstable
root@node1:~# cd /sys/fs/cgroup/kubepods.slice/kubepods-${PODS_QOS}.slice/kubepods-${PODS_QOS}-pod${PODS_UID}.slice/
root@node1:/sys/fs/cgroup/kubepods.slice/kubepods-burstable.slice/kubepods-burstable-pod9ea28e70_aa5b_49a2_81ce_c95ae0f6f5a5.slice# cat cri-containerd*/cgroup.procs | head -n 1
1243517
root@node1:/sys/fs/cgroup/kubepods.slice/kubepods-burstable.slice/kubepods-burstable-pod9ea28e70_aa5b_49a2_81ce_c95ae0f6f5a5.slice# cat cri-containerd*/cgroup.procs | tail -n 1
1243600
root@node1:/sys/fs/cgroup/kubepods.slice/kubepods-burstable.slice/kubepods-burstable-pod9ea28e70_aa5b_49a2_81ce_c95ae0f6f5a5.slice# PROCS_A=`cat cri-containerd*/cgroup.procs | head -n 1`
root@node1:/sys/fs/cgroup/kubepods.slice/kubepods-burstable.slice/kubepods-burstable-pod9ea28e70_aa5b_49a2_81ce_c95ae0f6f5a5.slice# PROCS_B=`cat cri-containerd*/cgroup.procs | tail -n 1`
root@node1:/sys/fs/cgroup/kubepods.slice/kubepods-burstable.slice/kubepods-burstable-pod9ea28e70_aa5b_49a2_81ce_c95ae0f6f5a5.slice# [[ ${PROCS_A} -gt ${PROCS_B} ]] && PROCS=${PROCS_A} || PROCS=${PROCS_B}
root@node1:/sys/fs/cgroup/kubepods.slice/kubepods-burstable.slice/kubepods-burstable-pod9ea28e70_aa5b_49a2_81ce_c95ae0f6f5a5.slice# echo $PROCS
1243600

```

以上每个`Pod`之所以有2个`container`，是因为每个`Pod`在创建的时候都先启动一个`pause`容器来实现共享网络和`Namespace`等一些。详见可以参考  [what is pause container](https://blog.devgenius.io/k8s-pause-container-f7abd1e9b488) 所以我们需要的业务容器的进程号是比`pause`容器的大，我们在比较二者的`PROCS_ID`大小之后取大的`PROCS_ID`

> **容器在Linux中都是以一个进程的方式运行的**

，现在我们拿到了业务容器的`PROCS_ID`, 通过`strace`方式查看该进程在后台的日志输出：

 ```shell
root@node1:/sys/fs/cgroup/kubepods.slice/kubepods-burstable.slice/kubepods-burstable-pod9ea28e70_aa5b_49a2_81ce_c95ae0f6f5a5.slice# strace -p $PROCS -f
strace: Process 1243600 attached with 6 threads
[pid 1243613] epoll_pwait(3, [], 128, 0, NULL, 0) = 0
[pid 1243613] epoll_pwait(3,  <unfinished ...>
[pid 1243616] futex(0xc000080148, FUTEX_WAIT_PRIVATE, 0, NULL <unfinished ...>
[pid 1243614] futex(0xc000038948, FUTEX_WAIT_PRIVATE, 0, NULL <unfinished ...>
[pid 1243615] futex(0xc000038d48, FUTEX_WAIT_PRIVATE, 0, NULL <unfinished ...>
[pid 1243612] restart_syscall(<... resuming interrupted read ...> <unfinished ...>
[pid 1243600] futex(0x8cb6a8, FUTEX_WAIT_PRIVATE, 0, NULL <unfinished ...>
[pid 1243613] <... epoll_pwait resumed>[], 128, 1703, NULL, 0) = 0
[pid 1243613] epoll_pwait(3,  <unfinished ...>
[pid 1243612] <... restart_syscall resumed>) = -1 ETIMEDOUT (Connection timed out)
[pid 1243613] <... epoll_pwait resumed>[], 128, 0, NULL, 0) = 0
[pid 1243613] futex(0x8cb6a8, FUTEX_WAKE_PRIVATE, 1 <unfinished ...>
[pid 1243612] epoll_pwait(3,  <unfinished ...>
[pid 1243613] <... futex resumed>)      = 1
[pid 1243612] <... epoll_pwait resumed>[], 128, 0, NULL, 0) = 0
[pid 1243600] <... futex resumed>)      = 0
[pid 1243600] epoll_pwait(3, [], 128, 0, NULL, 0) = 0
[pid 1243600] epoll_pwait(3,  <unfinished ...>
[pid 1243613] openat(AT_FDCWD, "port.txt", O_RDONLY|O_CLOEXEC <unfinished ...>
[pid 1243612] nanosleep({tv_sec=0, tv_nsec=20000},  <unfinished ...>
[pid 1243613] <... openat resumed>)     = -1 ENOENT (No such file or directory)
[pid 1243613] write(1, "Something went wrong...\n", 24 <unfinished ...>
....
 ```

以上的日志输出，可以看出 由于`port.txt`无法找到文件，那我们就需要进该进程的`namespace`查看该命名空间下的文件系统

```shell
root@node1:~# nsenter -t $PROCS -n ip addr show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
25: eth0@if26: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 9e:9d:46:60:a3:12 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.233.64.11/32 scope global eth0
       valid_lft forever preferred_lft forever
root@node1:~# nsenter -t $PROCS -n ss -tulpn
Netid                State                 Recv-Q                Send-Q                                 Local Address:Port                                 Peer Address:Port                Process                
tcp                  LISTEN                0                     4096                                         0.0.0.0:8080                                      0.0.0.0:*                    users:(("faultyapp",pid=1243600,fd=6))

```

以上的内容输出可以看出，该`faultapp`是监听在`8080`端口的上一个服务，现在我们就在`faultapp`的文件系统里创建`port.txt` 其内容为`8080`

```shell
root@node1:~# cd /proc/${PROCS}/root/
root@node1:/proc/1243600/root# ls
bin  dev  etc  proc  sys  var
root@node1:/proc/1243600/root# df -h .
Filesystem      Size  Used Avail Use% Mounted on
overlay          18G  9.5G  7.3G  57% /run/containerd/io.containerd.runtime.v2.task/k8s.io/b11e2212ca6892b6e4b617d5b033a5fc9670023092c16968ecde36b757a59d56/rootfs
root@node1:/proc/1243600/root# cd bin/
root@node1:/proc/1243600/root/bin# ls
faultyapp
root@node1:/proc/1243600/root/bin# echo -n "8080" > port.txt
root@node1:/proc/1243600/root/bin# ls -la
total 7124
drwxr-xr-x 1 root root    4096 Jan 19 09:05 .
drwxr-xr-x 1 root root    4096 Jan 17 12:55 ..
-rwxr-xr-x 1 root root 7279634 Jul 23 19:54 faultyapp
-rw-r--r-- 1 root root       4 Jan 19 09:05 port.txt
```

接下来再看下Kubernetes里该Pod的日志

```shell
root@node1:~# kubectl logs faultyapp-56bbb4bff6-8qjr2
Starting HTTP ...
Something went wrong...
Something went wrong...
Something went wrong...
Something went wrong...
It works! :)
It works! :)
It works! :)
It works! :)
It works! :)
It works! :)
It works! :)
...
```

### Bash Script

```shell
#!/bin/bash
#

debugpod(){
	local namespace=$1
	shift
	local pods=("$@")

	if [ ${#pods[@]} -eq 0 ]; then
		echo "please input pods name on this node.   Usage: debugpod NAMESPACE(default) pods"
	else
		for pod in "${pods[@]}"; do 
			pod_uid=`kubectl get -n ${namespace} pod ${pod} -o jsonpath='{.metadata.uid}' | sed 's/-/_/g'`
			pod_qos=`kubectl get -n ${namespace} pod ${pod} -o jsonpath='{.status.qosClass}'| tr '[:upper:]' '[:lower:]'`
							     pod_path="/sys/fs/cgroup/kubepods.slice/kubepods-${pod_qos}.slice/kubepods-${pod_qos}-pod${pod_uid}.slice"

			procs_a=`cat ${pod_path}/cri-containerd-*/cgroup.procs | head -n 1`
			procs_b=`cat ${pod_path}/cri-containerd-*/cgroup.procs | tail -n 1`

			[[ ${procs_a} -gt ${procs_b} ]] && procs=${procs_a} || procs=${procs_b}

			strace -p $procs -f
		done
	fi
}



```

以上shell脚本可以将以上一系列操作合并

```shell
# Edit your desired .bashrc, .profile, etc. file and add source /home/<user>/debugpod.sh
root@node1:/tmp# source /tmp/debugpod.sh
# debug vector pod in namespace vector
root@node1:/tmp# debugpod vector vector-zbkzq
strace: Process 3312 attached with 7 threads
[pid  3446] restart_syscall(<... resuming interrupted read ...> <unfinished ...>
[pid  3442] futex(0x7fec692fbd18, FUTEX_WAIT_PRIVATE, 1, NULL <unfinished ...>
[pid  3366] epoll_wait(3,  <unfinished ...>
[pid  3365] futex(0x7fec6acf9d18, FUTEX_WAIT_PRIVATE, 1, NULL <unfinished ...>
[pid  3363] futex(0x7fec6b0fbd18, FUTEX_WAIT_PRIVATE, 1, NULL <unfinished ...>
[pid  3312] futex(0x7fec6bf9b558, FUTEX_WAIT_PRIVATE, 1, NULL <unfinished ...>
[pid  3364] futex(0x7fec6aefad18, FUTEX_WAIT_PRIVATE, 1, NULL <unfinished ...>
[pid  3366] <... epoll_wait resumed>[], 1024, 87) = 0
[pid  3366] epoll_wait(3, [], 1024, 17) = 0
[pid  3366] futex(0x7fec692fbd18, FUTEX_WAKE_PRIVATE, 1 <unfinished ...>
[pid  3442] <... futex resumed>)        = 0
...
```



通过以上的一些列调试并修复了有问题的应用程序。感谢您和我一起阅读到最后！拥抱云原生，不断探索，让容器化将您的应用推向新的高度！