---
layout:     post
title:     使用Argo CD在Kubernetes部署简单应用
subtitle:  A Practical Tutorial With Kubernetes and Argo CD
date:       2023-03-17
author:     J
catalog: true
tags:
    - ArgoCD
    - Kubernetes
---
在本文中，我们将探索`Gitops`之Argo CD的功能并使用它来运行一个简单的演示应用程序。
Argo CD 是“用于 Kubernetes 的声明式 GitOps 持续交付工具”。它可以监控您的源存储库状态并自动将更改部署到您的目标Kubernetes集群。

Kubernetes 协调Pod部署和管理任务。它启动您的Pod，在它们失败时替换它们，并跨集群中的计算节点扩展您的服务。

Kubernetes 作持续交付工作流程的一部分。在合并更新新代码到源存储库之后自动部署可确保更改在通过一致的管道后快速到达您的集群。

### 什么是 Argo CD
ArgoCD是一种使用 Kubernetes 设置持续交付的流行工具。它通过根据实时工作负载不断协调存储库的状态，自动将应用程序部署到 Kubernetes 集群中。

GitOps 模型是Argo 设计不可或缺的一部分。它使存储库成为应用程序所需状态的唯一真实来源。您的应用程序需要的所有 Kubernetes 清单、Kustomize 模板、Helm 图表和配置文件都应该提交到您的存储库。这些资源“声明”您的应用程序的成功部署是什么样子的。

Argo 将声明的状态与集群中实际运行的状态进行比较，然后应用正确的更改来解决任何差异。这个过程可以配置为自动运行，防止你的集群偏离你的存储库。Argo 会在出现差异时重新同步状态，例如在您手动运行 Kubectl 命令之后。

Argo 带有 CLI 和 Web UI。它支持多租户和多集群环境，与 SSO 提供商集成，生成审计跟踪，并可以实施复杂的部署策略，例如金丝雀部署和蓝/绿升级。它还提供集成回滚，因此您可以快速从部署失败中恢复。

###基于Push和Pull的 CI/CD比较

大多数CI/CD实施都依赖于Push行为。这需要你将集群连接到 CI/CD 平台，然后在Pipeline的语法中使用 Kubectl 和 Helm 等工具来应用 Kubernetes 相关操作。

Argo 是一个基于Pull的 CI/CD 系统。它在您的 Kubernetes 集群中运行，并从你的Repo中拉取源代码。Argo 随后会为您应用更改，而无需手动配置管道。

此模型比基于Push的工作流更安全。不必公开Kubernetes集群的 API Server或将 Kubernetes Secrets这一类凭证存储在 CI/CD 平台中。破坏源存储库Repo只会让攻击者访问您的代码，而不是在Kubernetes中的应用部署。

### Argo CD 核心概念

以下是一些需要熟悉的Argo CD的概念
- `Argo controller` Argo 的应用程序控制器是您安装在集群中的组件。它实现类似Kubernetes Operator模式来监控您的应用程序并将它们的状态与它们的Git 存储库声明的状态进行比较。

- `Application` Argo 应用程序是由Argo CD部署在Kubernetes资源。Argo CD将集群中应用程序的详细信息存储为包含的自定义资源定义 (CRD)的实例。
- `Live state` 实时状态是Kubernetes集群内应用程序的当前状态，例如创建的 Pod 数量和它们正在运行的图像。
- `Target state` 目标状态是 Git 存储库声明的状态版本。当存储库发生变化时，Argo 将应用将活动状态演变为目标状态的操作。
- `Refresh` 当 Argo 从您的存储库中获取目标状态时发生刷新。它会将更改与实时状态进行比较，但不一定在此阶段应用它们。
- `Sync` 同步是应用刷新发现的更改的过程。每个 Sync 都会将集群刷新成目标状态。


### 使用 Argo CD 部署Nginx到 Kubernetes

#### 创建应用程序的 GitHub 存储库
首先，前往 GitHub 并为您的应用创建一个新的存储库。之后，将你的 repo 克隆到你的机器上，准备好提交你的 Kubernetes 清单：

```shell
root@ubuntu:~# git clone https://github.com/JaeGerW2016/argocd-demo-for-practice.git

```
```shell
root@ubuntu:~/argocd-demo-for-practice# cat > deployment.yaml << EOF
> apiVersion: apps/v1
> kind: Deployment
> metadata:
>   name: nginx
>   namespace: argo-demo
>   labels:
>     app.kubernetes.io/name: nginx
> spec:
>   replicas: 1
>   selector:
>     matchLabels:
>       app.kubernetes.io/name: nginx
>   template:
>     metadata:
>       labels:
>         app.kubernetes.io/name: nginx
>     spec:
>       containers:
>       - name: nginx
>         image: nginx:latest
>         ports:
>           - name: http
>             containerPort: 80
> EOF

root@ubuntu:~/argocd-demo-for-practice# cat > service.yaml << EOF
> apiVersion: v1
> kind: Service
> metadata:
>   name: nginx
>   namespace: argo-demo
> spec:
>   type: LoadBalancer
>   selector:
>     app.kubernetes.io/name: nginx
>   ports:
>     - protocol: TCP
>       port: 80
>       targetPort: http
> EOF

root@ubuntu:~/argocd-demo-for-practice# cat > namespace.yaml << EOF
> apiVersion: v1
> kind: Namespace
> metadata:
>   name: argo-demo
> EOF
```
将您的以上3个文件的更改提交到您的github存储库，然后将它们推送到 GitHub：

```
# git add .
# git commit -m "Added initial Kubernetes YAML files"
# git push origin main

```
### Github
![-2023-03-17-15-34-19933cfb850e0df0b6.png](https://youjb.com/images/2023/03/17/-2023-03-17-15-34-19933cfb850e0df0b6.png)


### 获取 Argo CLI
Argo 的 CLI 允许您从终端与您的应用程序进行交互。稍后您将需要它来向您的 Argo 实例注册您的应用程序。

```
# wget https://github.com/argoproj/argo-cd/releases/download/v2.6.6/argocd-linux-amd64
# chmod +x argocd-linux-amd64
# mv argocd-linux-amd64 /usr/local/bin/argocd
```

### 在Kubernetes集群中安装 Argo

接下来，在您的 Kubernetes 集群中安装 Argo。这将添加 Argo CD API、控制器和自定义资源定义 (CRD)。

首先为 Argo 创建一个命名空间：
```
# kubectl create namespace argocd
namespace/argocd created
````
接下来，使用 Kubectl 将 Argo CD 的 YAML 清单应用到您的集群。您可以 先检查清单，然后再使用它来查看将要创建的资源。
```
# kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

```
Argo 的所有组件在您的集群中运行可能需要几秒钟的时间。通过使用 Kubectl 列出命名空间中的部署来监控进度。
argocd清单部署准备就绪后，您可以看到在argocd的namespace命名空间里看到以下资源：

```
root@node1:~# kubectl get all -n argocd
NAME                                                   READY   STATUS    RESTARTS      AGE
pod/argocd-application-controller-0                    1/1     Running   0             46h
pod/argocd-applicationset-controller-fddd74c8f-654q8   1/1     Running   3 (46h ago)   9d
pod/argocd-dex-server-7444bc65c9-t74qk                 1/1     Running   1 (46h ago)   46h
pod/argocd-notifications-controller-549cfd4845-tsk7g   1/1     Running   4 (46h ago)   9d
pod/argocd-redis-7dd84c47d6-9d8sk                      1/1     Running   3 (46h ago)   9d
pod/argocd-repo-server-6cd74dc49c-5v8r5                1/1     Running   0             46h
pod/argocd-server-9599976c8-cw5t8                      1/1     Running   4 (24h ago)   9d

NAME                                              TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
service/argocd-applicationset-controller          ClusterIP   10.233.4.31     <none>        7000/TCP,8080/TCP            9d
service/argocd-dex-server                         ClusterIP   10.233.35.161   <none>        5556/TCP,5557/TCP,5558/TCP   9d
service/argocd-metrics                            ClusterIP   10.233.36.247   <none>        8082/TCP                     9d
service/argocd-notifications-controller-metrics   ClusterIP   10.233.10.27    <none>        9001/TCP                     9d
service/argocd-redis                              ClusterIP   10.233.32.79    <none>        6379/TCP                     9d
service/argocd-repo-server                        ClusterIP   10.233.29.207   <none>        8081/TCP,8084/TCP            9d
service/argocd-server                             ClusterIP   10.233.53.97    <none>        80/TCP,443/TCP               9d
service/argocd-server-metrics                     ClusterIP   10.233.8.146    <none>        8083/TCP                     9d

NAME                                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/argocd-applicationset-controller   1/1     1            1           9d
deployment.apps/argocd-dex-server                  1/1     1            1           9d
deployment.apps/argocd-notifications-controller    1/1     1            1           9d
deployment.apps/argocd-redis                       1/1     1            1           9d
deployment.apps/argocd-repo-server                 1/1     1            1           9d
deployment.apps/argocd-server                      1/1     1            1           9d

NAME                                                         DESIRED   CURRENT   READY   AGE
replicaset.apps/argocd-applicationset-controller-fddd74c8f   1         1         1       9d
replicaset.apps/argocd-dex-server-7444bc65c9                 1         1         1       9d
replicaset.apps/argocd-notifications-controller-549cfd4845   1         1         1       9d
replicaset.apps/argocd-redis-7dd84c47d6                      1         1         1       9d
replicaset.apps/argocd-repo-server-6cd74dc49c                1         1         1       9d
replicaset.apps/argocd-server-9599976c8                      1         1         1       9d

NAME                                             READY   AGE
statefulset.apps/argocd-application-controller   1/1     9d
```
### 创建argocd的ingress

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server-http-ingress
  namespace: argocd
  annotations:
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
spec:
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: argocd-server
            port:
              name: https
    host: myargocd.k8s.io
  tls:
  - hosts:
    - myargocd.k8s.io
    secretName: argocd-secret
  ingressClassName: nginx
```


### Argo CD的UI


![-2023-03-17-16-01-47e2de15f41e3a41c8.png](https://youjb.com/images/2023/03/17/-2023-03-17-16-01-47e2de15f41e3a41c8.png)



### Argo CD的CLI连接

在已经安装Argo CLI的服务器上,开始kubernetes的port-forward的本地转发,这会将您的本地端口8080 重定向到 Argo Server的端口443。
```
# kubectl port-forward svc/argocd-server -n argocd 8080:443

# argocd login localhost:8080
'admin:login' logged in successfully
Context 'localhost:8080' updated
```

### 使用 Argo 部署您的应用程序
一切准备就绪，可以开始向 Argo 部署应用程序了！首先，运行以下 CLI 命令来注册您的应用程序：
```
$ argocd app create argo-demo \
  --repo https://github.com/JaeGerW2016/argocd-demo-for-practice.git \
  --path . \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace argo-demo
application 'argo-demo' created
```
让我们解开这里发生的事情：

- `--repo` 该标志指定 Git 存储库的 URL。
- `--path .`该标志指示 Argo 在您的存储库中的此路径内搜索 Kubernetes 清单、Helm 图表和其他可部署资产。此处使用是因为示例清单存储在存储库的根目录中。
- `--dest-serverkubernetes.default.svc`该标志指定要部署到的 Kubernetes 集群的 URL。您可以在部署到运行 Argo 的同一集群时使用。
- `--dest-namespace`设置您的应用程序将部署到的 Kubernetes 命名空间。这应该与您的资源上`metadata.namespace`设置的字段相匹配。

现在将在 Argo 中注册。您可以使用以下命令检索其详细信息：`argocd app list`

```
root@node1:~# argocd app list
NAME              CLUSTER                         NAMESPACE  PROJECT  STATUS     HEALTH   SYNCPOLICY  CONDITIONS  REPO                                                         PATH  TARGET
argocd/argo-demo  https://kubernetes.default.svc  argo-demo  default  OutOfSync  Missing  <none>      <none>      https://github.com/JaeGerW2016/argocd-demo-for-practice.git  .   
```
![-2023-03-17-16-20-471f7af38a61fe6898.png](https://youjb.com/images/2023/03/17/-2023-03-17-16-20-471f7af38a61fe6898.png)

### Application的第一次同步
该应用程序显示为“丢失”和“不同步”。创建应用程序不会自动将其同步到您的集群中。现在执行同步让 Argo 应用当前由你的 repo 内容定义的目标状态：

```
root@node1:~# argocd app sync argo-demo
TIMESTAMP                  GROUP        KIND   NAMESPACE                  NAME    STATUS    HEALTH        HOOK  MESSAGE
2023-03-17T16:23:22+08:00            Service   argo-demo                 nginx  OutOfSync  Missing              
2023-03-17T16:23:22+08:00   apps  Deployment   argo-demo                 nginx  OutOfSync  Missing              
2023-03-17T16:23:22+08:00          Namespace                         argo-demo  OutOfSync  Missing              
2023-03-17T16:23:22+08:00          Namespace                         argo-demo    Synced  Missing              
2023-03-17T16:23:23+08:00          Namespace   argo-demo             argo-demo   Running    Synced              namespace/argo-demo created
2023-03-17T16:23:23+08:00            Service   argo-demo                 nginx  OutOfSync  Missing              service/nginx created
2023-03-17T16:23:23+08:00   apps  Deployment   argo-demo                 nginx  OutOfSync  Missing              deployment.apps/nginx created
2023-03-17T16:23:24+08:00            Service   argo-demo                 nginx    Synced  Progressing              service/nginx created
2023-03-17T16:23:24+08:00   apps  Deployment   argo-demo                 nginx    Synced  Progressing              deployment.apps/nginx created

Name:               argocd/argo-demo
Project:            default
Server:             https://kubernetes.default.svc
Namespace:          argo-demo
URL:                https://localhost:8080/applications/argo-demo
Repo:               https://github.com/JaeGerW2016/argocd-demo-for-practice.git
Target:             
Path:               .
SyncWindow:         Sync Allowed
Sync Policy:        <none>
Sync Status:        Synced to  (a4962d3)
Health Status:      Progressing

Operation:          Sync
Sync Revision:      a4962d3d775421c86e8ad045103d166256b7b7da
Phase:              Succeeded
Start:              2023-03-17 16:23:21 +0800 CST
Finished:           2023-03-17 16:23:23 +0800 CST
Duration:           2s
Message:            successfully synced (all tasks run)

GROUP  KIND        NAMESPACE  NAME       STATUS   HEALTH       HOOK  MESSAGE
       Namespace   argo-demo  argo-demo  Running  Synced             namespace/argo-demo created
       Service     argo-demo  nginx      Synced   Progressing        service/nginx created
apps   Deployment  argo-demo  nginx      Synced   Progressing        deployment.apps/nginx created
       Namespace              argo-demo  Synced                      

```
同步结果显示在您的终端中。您应该看到命名空间、服务和部署对象都已同步到您的集群中，如上面的命令输出所示。所有三个对象的消息确认它们已成功创建。

重复命令以检查应用程序的新状态：`apps list`
![-2023-03-17-16-24-33cf7c32bde698da73.png](https://youjb.com/images/2023/03/17/-2023-03-17-16-24-33cf7c32bde698da73.png)


### 同步应用程序更新

现在让我们对我们的应用程序进行更改。修改您的字段，因此 Deployment 中现在有五个 Pod：`spec.replicas`

`deployment.yaml`

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: argo-demo
  labels:
    app.kubernetes.io/name: nginx
spec:
  replicas: 5
...
```

提交并推送您的更改：
```
# git add .
# git commit -m "Run 5 replicas"
# git push origin main
```
接下来，重复命令以将更改应用到集群。`argocd app sync argo-demo`

或者，您可以单击用户UI界面中的“同步”按钮。

Argo 从 repo 刷新你的应用程序的目标状态，然后采取行动来转换实时状态。Deployment 已重新配置，现在运行五个 Pod：
```
$ kubectl get deployment -n argo-demo
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
nginx   5/5     5            5           12m
```

### 启用自动同步
在您重复同步命令之前，对五个副本的更改不会应用。不过，Argo 可以自动从您的存储库同步更改，从而无需每次都发出命令。这完全自动化了您的交付工作流程。

您可以通过单击用户界面中的“应用程序详细信息”按钮并向下滚动到“同步策略”部分来激活应用程序的自动同步。单击“启用自动同步”按钮。
![](https://spacelift.io/_next/image?url=https%3A%2F%2Fspaceliftio.wpcomstaging.com%2Fwp-content%2Fuploads%2F2023%2F03%2Fargo-cd-Enabling-Auto-Sync.png&w=1920&q=75)

还可以通过运行以下命令使用 CLI 启用自动同步：

```
argocd app set argo-demo --sync-policy automated
```
要测试自动同步输出，请将该字段恢复为三个副本：`spec.replicas`

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: argo-demo
  labels:
    app.kubernetes.io/name: nginx
spec:
  replicas: 3
  ...

```
提交并推送更改：
```
# git add .
# git commit -m "Back to 3 replicas"
# git push origin main
```
这次 Argo 会自动检测存储库的状态变化。在推送提交后的几分钟内，您应该会看到 Deployment 缩减为 3 个副本：
```
# kubectl get deployment -n argo-demo
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
nginx   3/3     3            3           1m
```

默认情况下，自动同步每三分钟运行一次。如果您需要更频繁的部署，您可以通过修改Argo的ConfigMap来配置时间间隔。

### 管理您的应用程序

Argo 的 CLI 和网络应用程序提供了广泛的选项来管理和监控您的部署。虽然这些不在本初学者教程的范围内，但您可以开始探索CLI 命令和 UI 面板来控制您的应用程序。大多数功能都在两个接口中实现。
![-2023-03-17-16-36-43654388d96861ee64.png](https://youjb.com/images/2023/03/17/-2023-03-17-16-36-43654388d96861ee64.png)



##参考文档
https://www.opsmx.com/blog/configuring-private-git-repo-in-argo-cd-to-deploy-kubernetes-apps/
https://spacelift.io/blog/argocd
