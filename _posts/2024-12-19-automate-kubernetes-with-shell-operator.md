---
layout: mypost
title: Automate kubernetes pod monitoring with shell-operator
categories: [shell-operator, crossplane, Kubernetes]
---

## 背景

在传统的印象中, 开发operator的工作需要,需要熟悉云原生方面的知识,比如:Go语言以及Kubernetes内部运行组件结构和原理,这个复杂性会限制operator的开发应用 
现在我们可以使用`shell operator` 这个项目来简化`Operator`的开发,它不需要你对Go语言和Kubernetes内部原理要求特别高,只需要你对`shell script`和一些`Docker`有一定的熟悉就可以

## 项目目标: 实时监控Pod

设计和实现一个针对 Kubernetes Pod 的实时监控系统。该系统将跟踪 Pod 的生命周期事件，特别专注于实时检测 Pod 的创建和删除。检测到这些事件后，系统将立即向指定的 Slack 频道发送通知。

以下我们想要使用 shell-operator 的工作流

![shell operator work flow](https://miro.medium.com/v2/resize:fit:1100/format:webp/1*YlYQA4E1efRWQ0ofJsLrdg.png)

## Shell Operator：Kubernetes 事件监视器

Shell Operator 是一个开源项目，旨在简化 Kubernetes 工作流的自动化。它使用户能够通过使用熟悉的语言（如 Bash、Python 或 Perl）编写脚本来创建自定义的 operator 和 controller。

### Shell Operator 的主要功能

Shell Operator 提供了几个关键功能，使其成为 Kubernetes 自动化的宝贵工具：

- 事件处理：Shell Operator 可以监视各种 Kubernetes 事件，例如 Pod、部署、机密和自定义资源定义 (CRD) 等资源的创建、删除和更新。
- 脚本执行：检测到事件后，Shell Operator 会执行用户定义的脚本。这些脚本可以用任何语言编写，因此系统管理员和开发人员即使更习惯使用脚本语言（而不是 Go）也可以使用它。
- 易于使用：Shell Operator 允许用户使用脚本定义自动化逻辑，从而简化了创建 Kubernetes 操作符的过程。这使得那些可能不熟悉 Kubernetes 内部复杂性的人可以更轻松地构建自定义操作符和控制器。
- 灵活性：它可以运行放置在 hooks 目录中的任何二进制文件或脚本，为您响应 Kubernetes 事件的方式提供灵活性。

通过利用这些功能，Shell Operator 使用户能够高效、有效地自动化和简化他们的 Kubernetes 工作流程。


## 前提条件

- kubernetes cluster
- crossplane
- crossplane provider
- slack channel

## 开发shell operator hooks

shell operator hooks是可执行的shell脚本存放目录, 由kubernetes cluster中的特定事件触发 例如: 创建 更新和删除kubernetes资源(pod service or crd) 
shell脚本自动在 shell operator注册  并在指定事件发生时执行 

以下是关于如何编写shell脚本的示例,该脚本对Pod的创建事件进行监听并向slack特定频道发出通知

```
#!/usr/bin/env bash

# Define the secret details
SECRET_NAMESPACE="crossplane-system"
SECRET_NAME="webhook-secret"
SECRET_KEY="webhook-url"

if [[ $1 == "--config" ]]; then
 cat <<EOF
configVersion: v1
kubernetes:
- apiVersion: v1
  kind: Pod
  executeHookOnEvent: ["Added"]
EOF
else
 # Retrieve the webhook URL from the secret
 WEBHOOK_URL=$(kubectl get secret $SECRET_NAME -n $SECRET_NAMESPACE -o jsonpath="{.data.$SECRET_KEY}" | base64 -d)

 if [[ -z "$WEBHOOK_URL" ]]; then
  echo "Error: WEBHOOK_URL is empty or the secret does not exist."
  exit 1
 fi

 # Iterate over all events in the binding context
 for i in $(seq 0 $(($(jq length $BINDING_CONTEXT_PATH) - 1))); do
  podName=$(jq -r .[$i].object.metadata.name $BINDING_CONTEXT_PATH)
  podNamespace=$(jq -r .[$i].object.metadata.namespace $BINDING_CONTEXT_PATH)
  creationTimestamp=$(jq -r .[$i].object.metadata.creationTimestamp $BINDING_CONTEXT_PATH)
  image=$(jq -r .[$i].object.spec.containers[0].image $BINDING_CONTEXT_PATH)
  # Create a YAML for the CRD
  cat <<EOF | kubectl apply -f -
apiVersion: http.crossplane.io/v1alpha2
kind: DisposableRequest
metadata:
  name: slack-webhook-creation-$podName
spec:
  deletionPolicy: Orphan
  forProvider:
    url: $WEBHOOK_URL
    method: POST
    body: '{
      "channel": "#dev",
      "username": "shell-operator-hook",
      "text": "A new pod has been created. Podname: $podName Namespace: $podNamespace Image: $image ",
      "icon_url": "https://example.com/path/to/icon.png"
      }'
EOF

  echo "CRD created for created pod '${podName}' with webhook URL."
 done
fi

```

## 创建基于Crossplane的 http 请求以向 Slack 发送通知

当我们的 Kubernetes 集群中创建或删除 Pod 时，我们使用 Crossplane 的 HTTP 提供程序向 Slack 发送通知。

Crossplane 是一种基础设施管理工具，它允许我们使用 Kubernetes 自定义资源定义 (CRD) 来定义和管理云基础设施。通过利用 Crossplane，我们可以创建 HTTP 请求，以更结构化和更易于管理的方式向 Slack 发送通知。


### crossplane Installation


Add the Crossplane Helm repository 
Add the Crossplane repository with the helm repo add command.

```
helm repo add crossplane-stable https://charts.crossplane.io/stable

Update the local Helm chart cache with helm repo update.

helm repo update
```

Install the Crossplane Helm chart 
Install the Crossplane Helm chart with helm install.


```
helm install crossplane \
--namespace crossplane-system \
--create-namespace crossplane-stable/crossplane 

View the installed Crossplane pods with kubectl get pods -n crossplane-system.

kubectl get pods -n crossplane-system
NAME                                                             READY   STATUS    RESTARTS        AGE
crossplane-6b8546d66d-8vzkq                                      1/1     Running   0               6d23h
crossplane-rbac-manager-7fdb5746bd-k6mg9                         1/1     Running   1 (5d19h ago)   6d23h
```



### crossplane provider http


To install provider-http, you have two options:

Using the Crossplane CLI in a Kubernetes cluster where Crossplane is installed:

```
crossplane xpkg install provider xpkg.upbound.io/crossplane-contrib/provider-http:v1.0.6
```

Manually creating a Provider by applying the following YAML:

```
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-http
spec:
  package: "xpkg.upbound.io/crossplane-contrib/provider-http:v1.0.6"
```

```
apiVersion: http.crossplane.io/v1alpha1
kind: ProviderConfig
metadata:
  name: default
  namespace: crossplane-system
spec:
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: webhook-secret
      key: webhook-url
```

View the installed Crossplane provider http pods with kubectl get pods -n crossplane-system.

```
kubectl get pods -n crossplane-system
NAME                                                             READY   STATUS    RESTARTS        AGE
crossplane-6b8546d66d-8vzkq                                      1/1     Running   0               6d23h
crossplane-contrib-provider-http-3a6d510a4856-7758694bb5-xqrzv   1/1     Running   0               47h
crossplane-rbac-manager-7fdb5746bd-k6mg9                         1/1     Running   1 (5d19h ago)   6d23h
```

要向 Slack 发送通知，我们需要定义一个DisposableRequest资源。此资源指定 HTTP 请求的详细信息，包括 URL、方法和正文。以下是设置 HTTP 请求以向 Slack 发送通知的方法：

```

apiVersion: http.crossplane.io/v1alpha2
kind: DisposableRequest
metadata:
  name: slack-webhook-creation-$podName
spec:
  deletionPolicy: Orphan
  forProvider:
    url: $WEBHOOK_URL
    method: POST
    body: '{
      "channel": "#dev",
      "username": "shell-operator-hook",
      "text": "...",
      "icon_emoji": ":ghost:"
    }'
```

## 创建 shell-operator Docker镜像

定义了环境以及设置 Shell Operator 和拷贝自定义Hooks目录到镜像中的步骤

```
FROM flant/shell-operator:latest 
COPY hooks /hooks 
RUN chmod +x /hooks/*
```

下一步就是构建 Docker 镜像。使用以下命令构建镜像

```
docker build -t your-dockerhub-username/shell-operator-hooks:v1 .
docker push your-dockerhub-username/shell-operator-hooks:v1
```


## 部署shell operator到Kubernetes集群

生成secret

```
#define your owen slack channel url
kubectl create secret generic webhook-secret --from-literal=webhook-url=https://hooks.slack.com/services/xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

`shell-operator.yaml`
```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: monitor-pods-acc
  namespace: default
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: pod-manager-clusterrole
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: [""]
    resources: ["namespaces"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["http.crossplane.io"]
    resources: ["disposablerequests"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: monitor-pods-acc-clusterrolebinding
subjects:
  - kind: ServiceAccount
    name: monitor-pods-acc
    namespace: default
roleRef:
  kind: ClusterRole
  name: pod-manager-clusterrole
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: v1
kind: Pod
metadata:
  name: shell-operator
  namespace: default
spec:
  serviceAccountName: monitor-pods-acc
  containers:
    - name: shell-operator
      image: your-dockerhub-username/shell-operator-hooks:latest
      imagePullPolicy: Always

```

```
kubectl apply -f shell-operator.yaml
```


### 验证

```
kubectl logs shell-operator -n default

...
{"level":"info","logger":"shell-operator.hook-manager.hook.executor","msg":"CRD created for created pod 'nginx-pod-demo' with webhook URL.","binding":"kubernetes","event":"kubernetes","hook":"pod-create-call-slack.sh","output":"stdout","queue":"main","task":"HookRun","time":"2024-12-19T07:06:30Z"}
{"level":"info","logger":"shell-operator","msg":"Hook executed successfully","binding":"kubernetes","event":"kubernetes","hook":"pod-create-call-slack.sh","queue":"main","task":"HookRun","time":"2024-12-19T07:06:30Z"}
{"level":"info","logger":"shell-operator","msg":"queue task HookRun:main:kubernetes:pod-delete-call-slack.sh:kubernetes","binding":"kubernetes","event.id":"538e7421-d9b7-4a71-8894-24261381dea9","queue":"main","time":"2024-12-19T07:07:03Z"}
{"level":"info","logger":"shell-operator","msg":"Execute hook","binding":"kubernetes","event":"kubernetes","hook":"pod-delete-call-slack.sh","queue":"main","task":"HookRun","time":"2024-12-19T07:07:03Z"}
{"level":"info","logger":"shell-operator.hook-manager.hook.executor","msg":"disposablerequest.http.crossplane.io/slack-webhook-deletion-nginx-pod-demo created","binding":"kubernetes","event":"kubernetes","hook":"pod-delete-call-slack.sh","output":"stdout","queue":"main","task":"HookRun","time":"2024-12-19T07:07:03Z"}
{"level":"info","logger":"shell-operator.hook-manager.hook.executor","msg":"CRD created for delete pod 'nginx-pod-demo' with webhook URL.","binding":"kubernetes","event":"kubernetes","hook":"pod-delete-call-slack.sh","output":"stdout","queue":"main","task":"HookRun","time":"2024-12-19T07:07:03Z"}
{"level":"info","logger":"shell-operator","msg":"Hook executed successfully","binding":"kubernetes","event":"kubernetes","hook":"pod-delete-call-slack.sh","queue":"main","task":"HookRun","time":"2024-12-19T07:07:04Z"}
...


```


```
# kubectl get disposablerequests.http.crossplane.io 
NAME                                    READY   SYNCED   EXTERNAL-NAME                           AGE
slack-webhook-creation-nginx-pod-demo   True    True     slack-webhook-creation-nginx-pod-demo   62m
slack-webhook-deletion-nginx-pod-demo   True    True     slack-webhook-deletion-nginx-pod-demo   62m
# kubectl get provider
NAME                                   AGE   CONFIG-NAME   RESOURCE-KIND       RESOURCE-NAME
1f23365a-083f-42d7-8830-ffd1922552fc   63m   default       DisposableRequest   slack-webhook-creation-nginx-pod-demo
5e5590f8-8db5-4e51-ac4b-f66c8e1b8949   62m   default       DisposableRequest   slack-webhook-deletion-nginx-pod-demo
# kubectl get providerconfig
NAME      AGE
default   24h

```

![](https://hv.z.wiki/autoupload/20241219/TXzs/1436X275/screenshot-2024-12-19-15-59-07.png)

通过利用 Shell Operator，我们创建了自定义Hook来处理 Kubernetes 事件并向 Slack 发送通知，从而增强了集群的可观测性

我们结合 Crossplane 强大插件功能来进行基础设施管理，展示了 Kubernetes 生态系统中多种工具的无缝集成 我们不需要从头开始创建自己的控制器；而是利用现有的 Shell 脚本技能来设计自动化
