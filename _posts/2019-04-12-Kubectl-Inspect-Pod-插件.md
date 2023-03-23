---
title: Kubectl Inspect Pod 插件
date: 2019-04-12 03:05:59
tags:
---
虽然目前还没有像标题所示的`kubectl inspect` command，但kubectl现在有一个新重新设计的插件系统，它允许您轻松创建自己的检查命令并让它做任何你想做的事情
![](https://upload-images.jianshu.io/upload_images/3481257-f5a85f5654443600.gif?imageMogr2/auto-orient/strip)



正如您所看到的，我的kubectl inspect插件使您能够快速显示Kubernetes资源的YAML - 语法高亮！

我知道人们讨厌YAML，但我更喜欢通过使用像kubectl describe这样的东西来查看它的原始YAML来检查资源。使用语法高亮显示，阅读YAML非常简单。

我最近从cat（meow）升级到[bat](https://github.com/sharkdp/bat)（squeek）用于打印文件，但是让它使我的资源用颜色闪耀需要太多的击键。例如：
```
$ kubectl get pod mypod -o yaml | bat -l yaml -p
```

这是要输入的许多额外字符，但它值得，因为它带来了这个颜色漂亮的YAML：
![](https://upload-images.jianshu.io/upload_images/3481257-be3ed65fbc26e23e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

使用该命令一段时间后，我有一个想法，并意识到我可以创建一个新的kubectl插件，允许我只需键入：

```
$ kubectl inspect pod mypod
```
我花了不到30秒的时间来创建插件。所需要的只是在我的PATH中添加一个名为kubectl-inspect的文件，其中包含以下内容：
`kubectl-inspect`
```
#!/bin/bash
kubectl get "$@" -o yaml --export | bat -l yaml -p
```

>PS：新的kubectl插件系统可从v1.12.0-beta.1开始提供。
[使用插件扩展 kubectl](https://k8smeetup.github.io/docs/tasks/extend-kubectl/kubectl-plugins/)

 ```
root@k8s-master-1:~/.kube# mkdir -p plugins
root@k8s-master-1:~/.kube/plugins# tree
.
└── inspect
    └── plugin.yaml

1 directory, 1 file
```

```
root@k8s-master-1:~/.kube/plugins# cat inspect/plugin.yaml
name: "inspect" 
shortDesc: "Inspect Yaml Highlighting"
longDesc: "Inspect Kubernetes Resource's YAML Highlighting"
example: "kubectl inspect pod [PODNAME] -n [NAMESPACE]"
command: "kubectl-inspect"

#add kube-inspect to PATH
root@k8s-master-1:~/.kube/plugins# ls /usr/local/bin/
helm  istioctl  kubectl-inspect  kube-prompt

##show  plugins list
root@k8s-master-1:~# kubectl plugin list
The following kubectl-compatible plugins are available:

/usr/local/bin/kubectl-inspect


```

```
root@k8s-master-1:~# kubectl get pod
NAME                     READY   STATUS    RESTARTS   AGE
myapp-8696457765-668bn   1/1     Running   3          30d
myapp-8696457765-f87qm   1/1     Running   3          30d
myapp-8696457765-jzqwm   1/1     Running   1          15d
myapp-8696457765-x2xz7   1/1     Running   3          30d
myapp-8696457765-x72sg   1/1     Running   3          30d
root@k8s-master-1:~# kubectl inspect pod myapp-8696457765-668bn -n default
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  generateName: myapp-8696457765-
  labels:
    app: myapp
    pod-template-hash: "8696457765"
  ownerReferences:
  - apiVersion: apps/v1
    blockOwnerDeletion: true
    controller: true
    kind: ReplicaSet
    name: myapp-8696457765
    uid: d96e9ab7-409f-11e9-b31a-000c2954713e
  selfLink: /api/v1/namespaces/default/pods/myapp-8696457765-668bn
spec:
  containers:
  - image: 314315960/zero-downtime-tutorial:green
    imagePullPolicy: IfNotPresent
    lifecycle:
      preStop:
        exec:
          command:
          - /bin/sh
          - -c
          - rm /usr/share/nginx/html/healthy.html && sleep 10
    name: nginx
    ports:
    - containerPort: 80
      protocol: TCP
    readinessProbe:
      failureThreshold: 2
      httpGet:
        path: /healthy.html
        port: 80
        scheme: HTTP
      periodSeconds: 1
      successThreshold: 1
      timeoutSeconds: 1
    resources: {}
    terminationMessagePath: /dev/termination-log
    terminationMessagePolicy: File
    volumeMounts:
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: default-token-b46m8
      readOnly: true
  dnsPolicy: ClusterFirst
  enableServiceLinks: true
  nodeName: 192.168.2.11
  restartPolicy: Always
  schedulerName: default-scheduler
  securityContext: {}
  serviceAccount: default
  serviceAccountName: default
  terminationGracePeriodSeconds: 10
  volumes:
  - name: default-token-b46m8
    secret:
      defaultMode: 420
      secretName: default-token-b46m8
status:
  phase: Pending
  qosClass: BestEffort

```

