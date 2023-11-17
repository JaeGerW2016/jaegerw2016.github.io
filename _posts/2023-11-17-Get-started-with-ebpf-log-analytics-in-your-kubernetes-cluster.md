---
layout: mypost
title: 使用Tetragon在Kubernetes进行eBPF日志分析
categories: [Kubernetes,Tetragon,eBPF]
---

## 简介

Linux 内核一直是实现安全性和可观察性功能的最佳场所之一，但在实践中也非常困难，因为您无法向内核添加新功能。eBPF 通过在运行时安全地增强内核功能来改变这一点。eBPF 允许沙盒程序在 Linux 内核中执行，而无需更改内核源代码或需要重新启动。它在运行时扩展了 Linux 内核。

这意味着，现在 Linux 内核的强大功能触手可及。您可以编写可以在内核中执行的程序，并且无需更改内核源代码或需要重新启动即可完成此操作。

日志记录是这项新技术的主要受益者之一。您现在可以使用 eBPF 启用内核级日志可观察性 - 捕获网络访问、文件访问等事件。这是云原生应用程序的游戏规则改变者，因为它允许您深入了解应用程序，而无需更改应用程序代码。

在这篇文章中，我们将探讨Tetragon与 Parseable和Vector的集成。我们还将研究一个非常具体的用例，用于审计 Kubernetes 中的敏感文件访问记录和产生相应的告警信息。
![Tetragon与 Parseable和Vector的集成](https://www.parseable.io/blog/assets/images/ecosystem-a3f3466a47daf0e0902805c712ea56ec.png)

- Tetragon是 Cilium 的一个开源项目，它使用 eBPF 提供运行时安全性、深度可观察性和内核级透明性。
- Parseable是面向开发人员的轻量级高性能日志分析堆栈。
- Vector是用于构建可观测性管道的轻量级、超快速工具。


这是本文中使用的高级架构图

![high-level architecture](https://www.parseable.io/blog/assets/images/workflow-7c4360529cfb2f0b246d320f4bc68fd3.png)


在图中可以看出：
- Kubernetes 集群中的 Tetragon 将内核ebpf日志存储在本地
- 通过 Vector 将 Tetragon 日志发送到 Parseable
- 通过 Parseable 触发Alert产生告警信息

## 前提
首先，需要满足以下条件：
- 具有管理员访问权限的Kubernetes 集群 
- 本地安装Kubectl CLI 命令行工具

## 安装
### Tetragon
```
helm repo add cilium https://helm.cilium.io
helm repo update
helm install tetragon cilium/tetragon -n kube-system
```
稍等Tetragon的Daemonset的pod运行之后
```
root@node1:~# kubectl get ds tetragon -n kube-system
NAME       DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
tetragon   3         3         3       3            3           <none>          24h

#调试输出tetragon的后台日志信息
root@node1:~# kubectl logs -n kube-system -l app.kubernetes.io/name=tetragon -c export-stdout -f
{"process_exec":{"process":{"exec_id":"bm9kZTE6MTMzNzgwMDAwMDAwOjI2Nzk=","pid":2679,"uid":0,"cwd":"/","binary":"/speaker","arguments":"--port=7472 --log-level=info","flags":"procFS auid rootcwd","start_time":"2023-11-16T02:33:50.429221821Z","auid":4294967295,"pod":{"namespace":"metallb-system","name":"speaker-c2bdr","container":{"id":"containerd://c1268e875848a9e6975c26fbc02f503eb4d8cda2986c0e0ff5ed3017bad7be0a","name":"speaker","image":{"id":"quay.io/metallb/speaker@sha256:b4a5576a3cf5816612f54355804bdb83a2560ad4120691129a2e5bac5339ee0c","name":"quay.io/metallb/speaker:v0.13.12"},"start_time":"2023-11-16T02:33:50Z","pid":1},"pod_labels":{"app":"metallb","component":"speaker","controller-revision-hash":"6d4487bcfb","pod-template-generation":"1"},"workload":"speaker","workload_kind":"DaemonSet"},"docker":"c1268e875848a9e6975c26fbc02f503","parent_exec_id":"bm9kZTE6MzMxMzAwMDAwMDA6MTIwMQ==","tid":2679},"parent":{"exec_id":"bm9kZTE6MzMxMzAwMDAwMDA6MTIwMQ==","pid":1201,"uid":0,"cwd":"/run/containerd/io.containerd.runtime.v2.task/k8s.io/a8f283c4ddbe4ebfb72c0eb224bc06fb458e86d56c586a498c0cbb864a033d8d","binary":"/usr/local/bin/containerd-shim-runc-v2","arguments":"-namespace k8s.io -id a8f283c4ddbe4ebfb72c0eb224bc06fb458e86d56c586a498c0cbb864a033d8d -address /run/containerd/containerd.sock","flags":"procFS auid","start_time":"2023-11-16T02:32:09.779221865Z","auid":4294967295,"parent_exec_id":"bm9kZTE6MTQ3MDAwMDAwMDox","tid":1201}},"node_name":"node1","time":"2023-11-16T02:33:50.429221745Z"}
...
```

为了可以格式化输出日志，可以在本地安装Tetragon CLI：
```
curl -L https://github.com/cilium/tetragon/releases/latest/download/tetra-linux-amd64.tar.gz | tar -xz
sudo mv tetra /usr/local/bin
```
使用 tetra CLI 命令行输入日志事件信息：
```
root@node1:~# kubectl logs -n kube-system -l app.kubernetes.io/name=tetragon -c export-stdout -f | tetra getevents -o compact
🚀 process metallb-system/speaker-c2bdr /speaker --port=7472 --log-level=info 
🚀 process vector/vector-zbkzq /usr/bin/vector --config-dir /etc/vector/  
🚀 process default/dev-pod /usr/bin/cat /etc/passwd                       
💥 exit    default/dev-pod /usr/bin/cat /etc/passwd 0            
🚀 process default/dev-pod /usr/bin/cat /etc/passwd                       
💥 exit    default/dev-pod /usr/bin/cat /etc/passwd 0            
🚀 process default/dev-pod /usr/bin/cat /etc/shadow                       
💥 exit    ingress-nginx/ingress-nginx-controller-855dc99c4-r9xkh /usr/bin/nginx -c /tmp/nginx/nginx-cfg1246894647 -t 0 
🚀 process ingress-nginx/ingress-nginx-controller-855dc99c4-r9xkh /usr/bin/nginx -c /tmp/nginx/nginx-cfg1829898433 -t 
💥 exit    ingress-nginx/ingress-nginx-controller-855dc99c4-r9xkh /usr/bin/nginx -c /tmp/nginx/nginx-cfg1829898433 -t 0 
💥 exit    default/dev-pod /usr/bin/cat /etc/shadow 0            
🚀 process default/dev-pod /usr/bin/cat /etc/group                        
💥 exit    default/dev-pod /usr/bin/cat /etc/group 0             
🚀 process default/dev-pod /usr/bin/cat /etc/gshadow                      
💥 exit    default/dev-pod /usr/bin/cat /etc/gshadow 0           
🚀 process ingress-nginx/ingress-nginx-controller-855dc99c4-r9xkh /usr/bin/nginx -c /etc/nginx/nginx.conf -s reload 
💥 exit    ingress-nginx/ingress-nginx-controller-855dc99c4-r9xkh /usr/bin/nginx -c /etc/nginx/nginx.conf -s reload 0 
💥 exit    ingress-nginx/ingress-nginx-controller-855dc99c4-r9xkh /usr/local/nginx/sbin/nginx  0 
💥 exit    ingress-nginx/ingress-nginx-controller-855dc99c4-r9xkh /usr/local/nginx/sbin/nginx  0 
💥 exit    ingress-nginx/ingress-nginx-controller-855dc99c4-r9xkh /usr/local/nginx/sbin/nginx  0 
💥 exit    ingress-nginx/ingress-nginx-controller-855dc99c4-r9xkh /usr/local/nginx/sbin/nginx  0 
💥 exit    ingress-nginx/ingress-nginx-controller-855dc99c4-r9xkh /usr/local/nginx/sbin/nginx  0 

```
### Vector

```
helm repo add vector https://helm.vector.dev
wget https://www.parseable.io/blog/vector/vector-tetragon-values.yaml
helm install vector vector/vector --namespace vector --create-namespace --values vector-tetragon-values.yaml
```

检查Pod运行情况

```
root@node1:~# kubectl get pods -n vector
NAME           READY   STATUS    RESTARTS   AGE
vector-t2mfs   1/1     Running   0          24h
vector-x6cb5   1/1     Running   0          24h
vector-zbkzq   1/1     Running   0          24h

```
在`vector-tetragon-values.yaml`配置文件中可以看出，`Vector`将存储在`/var/run/cilium/tetragon/tetragon.log`文件中的事件发送到`Parseable`的`tetrademo`日志流。

### Parseable
#### Setup Parseable with Local Storage
```
cat << EOF > parseable-env-secret
addr=0.0.0.0:8000
staging.dir=./staging
fs.dir=./data
username=admin
password=admin
EOF

kubectl create ns parseable
kubectl create secret generic parseable-env-secret --from-env-file=parseable-env-secret -n parseable

helm repo add parseable https://charts.parseable.io
helm install parseable parseable/parseable -n parseable --set "parseable.local=true"
```
#### parseable ingress yaml
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-parseable
spec:
  rules:
  - host: parseable-ui.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: parseable
            port:
              number: 80
  ingressClassName: nginx
```
### 创建Pod进行敏感文件访问
#### 创建具有特权访问权限的Pod
```
# dev-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: dev-pod
spec:
  containers:
    - command:
        - /nsenter
        - --all
        - --target=1
        - --
        - su
        - "-"
      image: alexeiled/nsenter:2.34
      name: nsenter
      securityContext:
        privileged: true
      stdin: true
      tty: true
  hostNetwork: true
  hostPID: true
```

### 创建TracingPolicy

#### 

```
apiVersion: cilium.io/v1alpha1
kind: TracingPolicy
metadata:
  name: "file-monitoring-filtered"
spec:
  kprobes:
  - call: "security_file_permission"
    syscall: false
    return: true
    args:
    - index: 0
      type: "file" # (struct file *) used for getting the path
    - index: 1
      type: "int" # 0x04 is MAY_READ, 0x02 is MAY_WRITE
    returnArg:
      index: 0
      type: "int"
    returnArgAction: "Post"
    selectors:
    - matchArgs:      
      - index: 0
        operator: "Equal"
        values:
        - "/etc/passwd" # the files that we care
      - index: 1
        operator: "Equal"
        values:
        - "2" # MAY_WRITE
  - call: "security_mmap_file"
    syscall: false
    return: true
    args:
    - index: 0
      type: "file" # (struct file *) used for getting the path
    - index: 1
      type: "uint32" # the prot flags PROT_READ(0x01), PROT_WRITE(0x02), PROT_EXEC(0x04)
    - index: 2
      type: "nop" # the mmap flags (i.e. MAP_SHARED, ...)
    returnArg:
      index: 0
      type: "int"
    returnArgAction: "Post"
    selectors:
    - matchArgs:
      - index: 0
        operator: "Equal"
        values:
        - "/etc/passwd" # the files that we care
      - index: 1
        operator: "Mask"
        values:
        - "2" # PROT_WRITE
  - call: "security_path_truncate"
    syscall: false
    return: true
    args:
    - index: 0
      type: "path" # (struct path *) used for getting the path
    returnArg:
      index: 0
      type: "int"
    returnArgAction: "Post"
    selectors:
    - matchArgs:
      - index: 0
        operator: "Equal"
        values:
        - "/etc/passwd" # the files that we care

```
该策略监控特定文件/etc/passwd。如果您检查TracingPolicy文件的内容，它会挂钩内核函数，通过以下函数中可以访问该文件：
- security_file_permission
- security_mmap_file
- security_path_truncate


### 从Pod访问敏感文件
```
kubectl exec -it dev-pod -n default -- cat /etc/passwd
kubectl exec -it dev-pod -n default -- cat /etc/shadow
kubectl exec -it dev-pod -n default -- cat /etc/group
kubectl exec -it dev-pod -n default -- cat /etc/gshadow
```

### 在Parseable设置Alert告警

```
{
  "version": "v1",
  "alerts": [
    {
      "targets": [
        {
          "type": "webhook",
          "endpoint": "https://webhook.site/4398aceb-376c-44aa-8c26-d378c5053cc5",
          "headers": {},
          "skip_tls_check": false,
          "repeat": {
            "interval": "10s",
            "times": 5
          }
        }
      ],
      "name": "Unauthorised access /etc/passwd",
      "message": "/etc/passwd file is accessed by an unauthorised pod",
      "rule": {
        "type": "composite",
        "config": "process_exec_process_arguments =% \"/etc/passwd\""
      }
    },
    {
      "targets": [
        {
          "type": "webhook",
          "endpoint": "https://webhook.site/4398aceb-376c-44aa-8c26-d378c5053cc5",
          "headers": {},
          "skip_tls_check": false,
          "repeat": {
            "interval": "10s",
            "times": 5
          }
        }
      ],
      "name": "Unauthorised access /etc/shadow",
      "message": "/etc/shadow file is accessed by an unauthorised pod",
      "rule": {
        "type": "composite",
        "config": "process_exec_process_arguments =% \"/etc/shadow\""
      }
    },
    {
      "targets": [
        {
          "type": "webhook",
          "endpoint": "https://webhook.site/4398aceb-376c-44aa-8c26-d378c5053cc5",
          "headers": {},
          "skip_tls_check": false,
          "repeat": {
            "interval": "10s",
            "times": 5
          }
        }
      ],
      "name": "Unauthorised access /etc/group",
      "message": "/etc/group file is accessed by an unauthorised pod",
      "rule": {
        "type": "composite",
        "config": "process_exec_process_arguments =% \"/etc/group\""
      }
    },
    {
      "targets": [
        {
          "type": "webhook",
          "endpoint": "https://webhook.site/4398aceb-376c-44aa-8c26-d378c5053cc5",
          "headers": {},
          "skip_tls_check": false,
          "repeat": {
            "interval": "10s",
            "times": 5
          }
        }
      ],
      "name": "Unauthorised access /etc/gshadow",
      "message": "/etc/gshadow file is accessed by an unauthorised pod",
      "rule": {
        "type": "composite",
        "config": "process_exec_process_arguments =% \"/etc/gshadow\""
      }
    }
  ]
}
```
### Webhook site Alert信息
![2023111715562719fbb155d72ff35f.png](https://youjb.com/images/2023/11/17/2023111715562719fbb155d72ff35f.png)


### Parseable UI state
![20231117155831a173364c3cb329b4.png](https://youjb.com/images/2023/11/17/20231117155831a173364c3cb329b4.png)

## 参考文档
[https://ebpf.io/what-is-ebpf/](https://ebpf.io/what-is-ebpf/)

[https://tetragon.io/docs/](https://tetragon.io/docs/)

[https://www.parseable.io/docs](https://www.parseable.io/docs)

[https://vector.dev/docs/](https://vector.dev/docs/)