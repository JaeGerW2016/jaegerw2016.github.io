---
layout:     post
title:     Kubernetes 安全加固之Pod安全
subtitle:  Kubernetes Hardening Tutorial Pods
date:       2023-01-31
author:     J
catalog: true
tags:
    - Kubernetes
---

## Pod安全
Pod 是最小的可部署 Kubernetes 单元，由一个或多个容器组成。Pod 通常是网络攻击者在利用容器时的初始执行环境。出于这个原因，Pod 应该被加固以使利用更加困难并限制成功妥协的影响。

## 不要以root用户身份运行容器
默认情况下，Docker 容器将以 root 用户身份运行，而 K8s 将允许它这样做。

查看以下Dockerfile
```
FROM golang:1.19-buster as builder
ENV GO111MODULE on

WORKDIR /go/release
COPY . .
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -a -installsuffix cgo -o app .

FROM alpine:3.12
WORKDIR /
COPY --from=builder /go/release/app .
ENTRYPOINT ["/app"]

EXPOSE 8080/tcp

```
查看deployment的yaml文件

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: k8s-security-demo
  namespace: default
  labels:
    app: k8s-security-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: k8s-security-demo
  template:
    metadata:
      labels:
        app: k8s-security-demo
    spec:
      containers:
      - name: hello-server
        image: 314315960/k8s-security-demo:pod-as-root
        ports:
          - containerPort: 8080
---
kind: Service
apiVersion: v1
metadata:
  name: k8s-security-demo
  namespace: default
spec:
  selector:
    app: k8s-security-demo
  ports:
    - port: 80
      targetPort: 8080

```


在Dockerfile中没有明确声明`USER`命令的情况下,该容器会以`root`用户来运行

```
root@node1:~/kubernetes-security-demo# kubectl exec -it k8s-security-demo-58866bd747-pdfzt -- /bin/sh
/ #
/ # whoami
root

```

因此就需要在Dockerfile中声明一个用户:

```
RUN adduser -D myuser
USER myuser
```
以及在deployment的yaml声明:

```
...
securityContext:
  runAsNonRoot: True
  runAsUser: 1000
...

```

优化后的Dockerfile

```
FROM golang:1.19-buster as builder
ENV GO111MODULE on

WORKDIR /go/release
COPY . .
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -a -installsuffix cgo -o app .

FROM alpine:3.12
RUN adduser -D myuser
USER myuser
WORKDIR /
COPY --from=builder /go/release/app .
ENTRYPOINT ["/app"]

EXPOSE 8080/tcp
```
优化后的yaml文件

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: k8s-security-demo
  namespace: default
  labels:
    app: k8s-security-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: k8s-security-demo
  template:
    metadata:
      labels:
        app: k8s-security-demo
    spec:
      containers:
      - name: hello-server
        image: 314315960/k8s-security-demo:pod-as-non-root
        imagePullPolicy: Always
        ports:
          - containerPort: 8080
        securityContext:
          runAsNonRoot: True
          runAsUser: 1000
---
kind: Service
apiVersion: v1
metadata:
  name: k8s-security-demo
  namespace: default
spec:
  selector:
    app: k8s-security-demo
  ports:
    - port: 80
      targetPort: 8080
```

验证


```
root@node1:~/kubernetes-security-demo# kubectl exec -it k8s-security-demo-67c4598948-gzp4k -- /bin/sh
/ $
/ $ whoami
myuser

```

## 授权只读文件系统
在我们以Root身份运行的容器中,我们可以在root文件系统上创建任何文件

```
root@node1:~/kubernetes-security-demo# kubectl exec -it k8s-security-demo-58866bd747-wxhhc -- /bin/sh
/ # cd /tmp
/tmp # echo 'kubernetes security demo' > file.txt
/tmp # cat file.txt
kubernetes security demo
```
要授权只读的root文件系统，我们只需在`securityContext`添加一行：
```
readOnlyRootFilesystem: True
```
重新部署deployment之后后，如果我们运行另一个测试：

```
root@node1:~/kubernetes-security-demo# kubectl exec -it k8s-security-demo-648f877764-zbpcz -- /bin/sh
/ # cd /tmp
/tmp # echo 'kubernetes security demo' > file.txt
/bin/sh: can't create file.txt: Read-only file system
```

## trivy图像扫描

今天，让我们快速了解一个镜像扫描的工具：[trivy](https://github.com/aquasecurity/trivy)

```
root@ubuntu:/tmp# trivy image 314315960/k8s-security-demo:pod-as-root
2023-02-01T11:01:18.631+0800    INFO    Vulnerability scanning is enabled
2023-02-01T11:01:18.632+0800    INFO    Secret scanning is enabled
2023-02-01T11:01:18.632+0800    INFO    If your scanning is slow, please try '--security-checks vuln' to disable secret scanning
2023-02-01T11:01:18.632+0800    INFO    Please see also https://aquasecurity.github.io/trivy/v0.36/docs/secret/scanning/#recommendation for faster secret detection
2023-02-01T11:01:19.172+0800    INFO    Detected OS: alpine
2023-02-01T11:01:19.172+0800    INFO    Detecting Alpine vulnerabilities...
2023-02-01T11:01:19.174+0800    INFO    Number of language-specific files: 0
2023-02-01T11:01:19.174+0800    WARN    This OS version is no longer supported by the distribution: alpine 3.12.12
2023-02-01T11:01:19.175+0800    WARN    The vulnerability detection may be insufficient because security updates are not provided

314315960/k8s-security-demo:pod-as-root (alpine 3.12.12)

Total: 1 (UNKNOWN: 0, LOW: 0, MEDIUM: 0, HIGH: 0, CRITICAL: 1)

┌─────────┬────────────────┬──────────┬───────────────────┬───────────────┬─────────────────────────────────────────────────────────────┐
│ Library │ Vulnerability  │ Severity │ Installed Version │ Fixed Version │                            Title                            │
├─────────┼────────────────┼──────────┼───────────────────┼───────────────┼─────────────────────────────────────────────────────────────┤
│ zlib    │ CVE-2022-37434 │ CRITICAL │ 1.2.12-r0         │ 1.2.12-r2     │ zlib: heap-based buffer over-read and overflow in inflate() │
│         │                │          │                   │               │ in inflate.c via a...                                       │
│         │                │          │                   │               │ https://avd.aquasec.com/nvd/cve-2022-37434                  │
└─────────┴────────────────┴──────────┴───────────────────┴───────────────┴─────────────────────────────────────────────────────────────┘
```
可以看出`FROM alpine:3.12`这个基础镜像存在`CRITICAL`级别的漏洞,所以我们需要升级我们的基础镜像为`alpine:3.17`之后在用`trivy`做一次镜像漏洞扫描

```
root@ubuntu:~/kubernetes-security-demo/pod-run-as-root# trivy image 314315960/k8s-security-demo:pod-as-root
2023-02-01T11:07:09.715+0800    INFO    Vulnerability scanning is enabled
2023-02-01T11:07:09.715+0800    INFO    Secret scanning is enabled
2023-02-01T11:07:09.715+0800    INFO    If your scanning is slow, please try '--security-checks vuln' to disable secret scanning
2023-02-01T11:07:09.715+0800    INFO    Please see also https://aquasecurity.github.io/trivy/v0.36/docs/secret/scanning/#recommendation for faster secret detection
2023-02-01T11:07:10.080+0800    INFO    Detected OS: alpine
2023-02-01T11:07:10.081+0800    INFO    Detecting Alpine vulnerabilities...
2023-02-01T11:07:10.087+0800    INFO    Number of language-specific files: 0

314315960/k8s-security-demo:pod-as-root (alpine 3.17.1)

Total: 0 (UNKNOWN: 0, LOW: 0, MEDIUM: 0, HIGH: 0, CRITICAL: 0)
```

## Pod Security Admission

Kubernetes Pod 安全标准为 Pod 定义了不同的隔离级别。这些标准允许您以清晰、一致的方式定义您希望如何限制 pod 的行为。

Kubernetes 提供了一个内置的Pod Security 准入控制器执行 Pod 安全标准。Pod 安全限制适用于命名空间创建 Pod 时的级别。

详见官方文档关于[Pod Security Admission](https://kubernetes.io/docs/concepts/security/pod-security-admission/)部分

## OPA

Open Policy Agent（OPA，发音为“oh-pa”）是一个开源的通用策略引擎，它统一了堆栈中的策略执行。OPA 提供了一种高级声明性语言，可让您将策略指定为代码和简单的 API，以从您的软件中卸载策略决策制定。您可以使用 OPA 在微服务、Kubernetes、CI/CD 管道、API 网关等中实施策略。

详见官方文档[Open Policy Agent](https://www.openpolicyagent.org/docs/latest/#overview)部分
