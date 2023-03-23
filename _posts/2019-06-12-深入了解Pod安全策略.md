---
title: 深入了解Pod安全策略
date: 2019-06-12 09:29:00
tags: kubernetes
---
###  什么是Pod安全策略

Pod安全策略是群集范围的资源，用于控制pod规范的安全敏感方面。PSP对象定义了一组必须运行的条件才能被接受到系统中，以及相关字段的默认值。PodSecurityPolicy是一个可选的许可控制器，默认情况下通过API启用，因此可以在未启用PSP许可插件的情况下部署策略。它同时用作验证和变异控制器。

**Pod安全策略允许您控制：**

- 特权容器的运行
- 使用主机名称空间
- 使用主机网络和端口
- 卷类型的用法
- 主机文件系统的用法
- Flexvolume驱动程序的白名单
- 拥有pod卷的FSGroup的分配
- 使用只读根文件系统的要求
- 容器的用户和组ID
- 根权限的升级
- Linux功能，SELinux上下文，AppArmor，seccomp，sysctl配置文件

#### 检查并查看PSP准入控制器是否已启用

**kube-apiserver pod形式**

```shell
#此插件负责在创建和修改 pod 时根据请求的安全上下文和可用的 pod 安全策略确定是否应该通过 pod
grep -- --enable-admission-plugins /etc/kubernetes/manifests/kube-apiserver.yaml
   - --enable-admission-plugins=AlwaysPullImages,DenyEscalatingExec,EventRateLimit,NodeRestriction,ServiceAccount,PodSecurityPolicy

```

**kube-apiserver 二进制形式**

```shell
# --admission-control在1.10中已弃用并替换为--enable-admission-plugins
# 此插件负责在创建和修改 pod 时根据请求的安全上下文和可用的 pod 安全策略确定是否应该通过 pod
--enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota,NodeRestriction,MutatingAdmissionWebhook,ValidatingAdmissionWebhook,PodSecurityPolicy
```

### 使用的受限制PodSecurityPolicy的示例

首先，让我们创建一个示例部署，如下所示。在这个清单中，还有一些`securityContext`在`pod`和`container`级别定义。

- `runAsUser: 1000` 表示pod中的所有容器都将以用户UID 1000运行
- `fsGroup: 2000` 表示已挂载卷的所有者，及该卷中创建的任何文件GID都是2000
- `allowPrivilegeEscalation: false` 表示容器无法升级权限
- `readOnlyRootFilesystem: true` 表示容器只能读取根文件系统

*example-restricted-deployment.yaml:*

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: alpine-restricted
  labels:
    app: alpine-restricted
spec:
  replicas: 1
  selector:
    matchLabels:
      app: alpine-restricted
  template:
    metadata:
      labels:
        app: alpine-restricted
    spec:
      securityContext:
        runAsUser: 1000
        fsGroup: 2000
      volumes:
      - name: sec-ctx-vol
        emptyDir: {}
      containers:
      - name: alpine-restricted
        image: alpine:3.9
        command: ["sleep", "3600"]
        volumeMounts:
        - name: sec-ctx-vol
          mountPath: /data/demo
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true

```

```shell
root@k8s-master-1:~/k8s_manifests/example-podsecuritypolicy# kubectl get deploy,pod,rs
NAME                                         READY   UP-TO-DATE   AVAILABLE   AGE
deployment.extensions/alpine-restricted      0/1     0            0           2m46s

NAME                                        READY   STATUS    RESTARTS   AGE


NAME                                                    DESIRED   CURRENT   READY   AGE
replicaset.extensions/alpine-restricted-7cddf6f875      1         0         0       2m46s

root@k8s-master-1:~/k8s_manifests/example-podsecuritypolicy# kubectl describe replicaset.extensions/alpine-restricted-7cddf6f875 | tail -n 3
  Type     Reason        Age                   From                   Message
  ----     ------        ----                  ----                   -------
  Warning  FailedCreate  18s (x17 over 5m45s)  replicaset-controller  Error creating: pods "alpine-restricted-7cddf6f875-" is forbidden: no providers available to validate pod request

```

![](https://i.loli.net/2019/06/12/5d006aeaeb21678351.png)

如果没有Pod安全策略，replicaset控制器无法创建pod。让我们部署一个受限制的**PSP**并创建一个`ClusterRole`和`ClusterRolebinding`，这将允许replicaset集控制器使用上面定义的PSP。

*PSP-restricted.yaml：*

```yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: psp.restricted
  annotations:
    seccomp.security.alpha.kubernetes.io/defaultProfileName:  'runtime/default'
    seccomp.security.alpha.kubernetes.io/allowedProfileNames: 'runtime/default'
spec:
  readOnlyRootFilesystem: true
  privileged: false
  allowPrivilegeEscalation: false
  runAsUser:
    rule: 'MustRunAsNonRoot'
  supplementalGroups:
    rule: 'MustRunAs'
    ranges:
      - min: 1
        max: 65535
  fsGroup:
    rule: 'MustRunAs'
    ranges:
      - min: 1
        max: 65535
  seLinux:
    rule: 'RunAsAny'
  volumes:
  - configMap
  - emptyDir
  - secret
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: psp:restricted
rules:
- apiGroups:
  - policy
  resourceNames:
  - psp.restricted
  resources:
  - podsecuritypolicies
  verbs:
  - use
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: psp:restricted:binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: psp:restricted
subjects:
  - kind: ServiceAccount
    name: replicaset-controller
    namespace: kube-system

```



在`psp-restricted.yaml`有与Pod限制`psp.restricted`策略：

- 根文件系统必须是只读的
- 它不允许特权容器
- 特权升级是被禁止的 `readOnlyRootFilesystem: true`
- 用户必须是非根的 `runAsUser: rule: 'MustRunAsNonRoot'`
- fsGroup和supplementalGroups不能是root `defined allowed range`
- selinux的*RunAsAny* 允许任何`seLinuxOptions`指定
- pod只能使用指定的卷：`configMap`，`emptyDir`和`secret`
- 在注释部分有一个`seccomp`配置文件，由容器使用。由于这些注释，PSP将在部署之前改变podspec。如果我们检查Kubernetes中的pod清单，我们也会在那里看到`seccomp`注释。

> 可以通过PodSecurityPolicy上的注释来控制在pod中使用seccomp配置文件。Seccomp是Kubernetes中的alpha功能。
>
> **seccomp.security.alpha.kubernetes.io/defaultProfileName** - 指定要应用于容器的默认seccomp配置文件的注释。可能的值是：
>
> - `unconfined` - 如果没有提供替代方案，则Seccomp不会应用于容器进程（这是Kubernetes中的默认值）。
> - `runtime/default` - 使用默认的容器运行时配置文件。
> - `docker/default` - 使用Docker默认的seccomp配置文件。自Kubernetes 1.11起已弃用。请`runtime/default`改用。
> - `localhost/<path>`- 将配置文件指定为位于其上的节点上的文件 `<seccomp_root>/<path>`，其中`<seccomp_root>`通过`--seccomp-profile-root`Kubelet 上的标志定义 。
>
> **seccomp.security.alpha.kubernetes.io/allowedProfileNames** - 指定pod seccomp注释允许哪些值的注释。指定为逗号分隔的允许值列表。可能的值是上面列出的值，以及`*`允许所有配置文件。缺少此注释意味着无法更改默认值。

在`psp:restricted`ClusterRole和`psp:restricted:binding`ClusterRoleBinding中，我们`replicaset-controller`可以使用服务帐户的`psp.restricted`的PodSecurityPolicy策略。

现在，让我们删除`example-restricted-deployment.yaml`并再次应用它：

```shell
root@k8s-master-1:~/k8s_manifests/example-podsecuritypolicy# kubectl delete -f example-restricted-deployment.yaml 
deployment.apps "alpine-restricted" deleted
root@k8s-master-1:~/k8s_manifests/example-podsecuritypolicy# kubectl apply -f example-restricted-deployment.yaml 
deployment.apps/alpine-restricted created
root@k8s-master-1:~/k8s_manifests/example-podsecuritypolicy# kubectl get deploy,rs,pod
NAME                                         READY   UP-TO-DATE   AVAILABLE   AGE
deployment.extensions/alpine-restricted      0/1     1            0           10s

NAME                                                    DESIRED   CURRENT   READY   AGE
replicaset.extensions/alpine-restricted-7cddf6f875      1         1         0       10s

NAME                                        READY   STATUS              RESTARTS   AGE
pod/alpine-restricted-7cddf6f875-kmmdj      0/1     ContainerCreating   0          10s

root@k8s-master-1:~/k8s_manifests/example-podsecuritypolicy# kubectl get deploy,rs,pod
NAME                                         READY   UP-TO-DATE   AVAILABLE   AGE
deployment.extensions/alpine-restricted      1/1     1            1           2m1s

NAME                                                    DESIRED   CURRENT   READY   AGE
replicaset.extensions/alpine-restricted-7cddf6f875      1         1         1       2m1s

NAME                                        READY   STATUS    RESTARTS   AGE
pod/alpine-restricted-7cddf6f875-kmmdj      1/1     Running   0          2m1s

```

![](https://i.loli.net/2019/06/12/5d00712e6133c54809.png)

好的，让我们来看看`pod`注释：

```shell
root@k8s-master-1:~/k8s_manifests/example-podsecuritypolicy# kubectl get pod/alpine-restricted-7cddf6f875-kmmdj -o jsonpath='{.metadata.annotations}'
map[kubernetes.io/psp:psp.restricted seccomp.security.alpha.kubernetes.io/pod:runtime/default]
```

我们可以看到有两个注释集：

- kubernetes.io/psp:psp.restricted
- seccomp.security.alpha.kubernetes.io/pod:runtime/default

第一个是`PodSecurityPolicy`pod使用的。第二个是`seccomp`pod使用的配置文件。**Seccomp**（安全计算模式）是一种Linux内核功能，用于限制容器内可用的操作。

它真的有效吗？

您可以`sleep 3600`通过我们的alpine pod运行的流程状态在`node`中进行检查：

```shell
root@k8s-node-3:~# ps ax | grep "sleep 3600"
 68355 ?        Ss     0:00 sleep 3600
root@k8s-node-3:~# grep Seccomp /proc/68355/status
Seccomp:	2
```

>/ proc文件系统提供了一些[Seccomp](<http://man7.org/linux/man-pages/man2/seccomp.2.html>)信息：
>
```shell
Seccomp: Seccomp mode of the process (since Linux 3.8, see
                 seccomp(2)).  0 means SECCOMP_MODE_DISABLED; 1 means SEC‐
                 COMP_MODE_STRICT; 2 means SECCOMP_MODE_FILTER.  This field
                 is provided only if the kernel was built with the CON‐
                FIG_SECCOMP kernel configuration option enabled.
```

### 创建特权PodSecurityPolicy并将其与指定的服务帐户一起使用

`privileged-namespace.yaml`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: privileged
```



`example-privileged-deployment.yaml:`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: alpine-privileged
  namespace: privileged
  labels:
    app: alpine-privileged
spec:
  replicas: 1
  selector:
    matchLabels:
      app: alpine-privileged
  template:
    metadata:
      labels:
        app: alpine-privileged
    spec:
      securityContext:
        runAsUser: 1000
        fsGroup: 2000
      volumes:
      - name: sec-ctx-vol
        emptyDir: {}
      containers:
      - name: alpine-privileged
        image: alpine:3.9
        command: ["sleep", "1800"]
        volumeMounts:
        - name: sec-ctx-vol
          mountPath: /data/demo
        securityContext:
          allowPrivilegeEscalation: true
          readOnlyRootFilesystem: false

```



```shell
root@k8s-master-1:~/k8s_manifests/example-podsecuritypolicy# kubectl apply -f privileged-namespace.yaml 
namespace/privileged created
root@k8s-master-1:~/k8s_manifests/example-podsecuritypolicy# kubectl apply -f example-privileged-deployment.yaml 
deployment.apps/alpine-privileged created
root@k8s-master-1:~/k8s_manifests/example-podsecuritypolicy# kubectl get deploy,rs,pod -n privileged 
NAME                                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.extensions/alpine-privileged   0/1     0            0           19s

NAME                                                 DESIRED   CURRENT   READY   AGE
replicaset.extensions/alpine-privileged-66656f6765   1         0         0       19s

root@k8s-master-1:~/k8s_manifests/example-podsecuritypolicy# kubectl describe replicaset.extensions/alpine-privileged-66656f6765 -n privileged | tail -n 3
  Type     Reason        Age                  From                   Message
  ----     ------        ----                 ----                   -------
  Warning  FailedCreate  28s (x15 over 110s)  replicaset-controller  Error creating: pods "alpine-privileged-66656f6765-" is forbidden: unable to validate against any pod security policy: [spec.containers[0].securityContext.readOnlyRootFilesystem: Invalid value: false: ReadOnlyRootFilesystem must be set to true spec.containers[0].securityContext.allowPrivilegeEscalation: Invalid value: true: Allowing privilege escalation for containers is not allowed]
```

使用`replicaset-controller`服务帐户创建pod ，允许使用该`psp.restricted`策略。Pod创建失败，因为podspec包含`allowPrivilegeEscalation: true`和`readOnlyRootFilesystem: false` `securityContext`s - 此时，就这些而言，这些对于`psp.restricted`都是无效值。

![](https://i.loli.net/2019/06/12/5d00a69bf02b932883.png)

解决方法：

创建namespace:privileged 的serviceaccount: privileged-sa

```shell
root@k8s-master-1:~/k8s_manifests/example-podsecuritypolicy# cat privileged-serviceAccount.yaml 
piVersion: v1
kind: ServiceAccount
metadata:
  name: privileged-sa
  namespace: privileged
```



```yaml
kubectl apply -f privileged-serviceAccount.yaml
```



pod的deployment的spec.template.spec新增`serviceAccountName: privileged-sa`

```yaml
12...
13  template:
14    metadata:
15      labels:
16        app: alpine-privileged
17    spec:
18      serviceAccountName: privileged-sa #new add
19      securityContext:
20        runAsUser: 1
21        fsGroup: 1
22...
```

创建psp-privileged.yaml 及创建Role和Rolebinding与privilege-sa绑定

```
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: psp.privileged
spec:
  readOnlyRootFilesystem: false
  privileged: true
  allowPrivilegeEscalation: true
  runAsUser:
    rule: 'RunAsAny'
  supplementalGroups:
    rule: 'RunAsAny'
  fsGroup:
    rule: 'RunAsAny'
  seLinux:
    rule: 'RunAsAny'
  volumes:
  - configMap
  - emptyDir
  - secret
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: psp:privileged
  namespace: privileged
rules:
- apiGroups:
  - policy
  resourceNames:
  - psp.privileged
  resources:
  - podsecuritypolicies
  verbs:
  - use
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: psp:privileged:binding
  namespace: privileged
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: psp:privileged
subjects:
  - kind: ServiceAccount
    name: privileged-sa
    namespace: privileged
```

`psp.privileged`策略包含`readOnlyRootFilesystem: false`和`allowPrivilegeEscalation: true`。命名空间中privilegedprivileged`的`privileged-sa`服务帐户`允许我们使用`psp.privileged`策略，因此，如果我们部署修改后的`example-privileged-deployment.yaml`，可以正常启动pod。

现在让我们再次应用*example-privileged-deployment.yaml*：

```
root@k8s-master-1:~/k8s_manifests/example-podsecuritypolicy# kubectl get deploy,rs,pod -n privileged 
NAME                                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.extensions/alpine-privileged   1/1     1            1           10m

NAME                                                 DESIRED   CURRENT   READY   AGE
replicaset.extensions/alpine-privileged-75cf8fcc55   1         1         1       10m

NAME                                     READY   STATUS    RESTARTS   AGE
pod/alpine-privileged-75cf8fcc55-cgjfk   1/1     Running   0          10m

```

![](https://i.loli.net/2019/06/12/5d00ae035adab38018.png)

```shell
#检查rs启用serviceAccount
root@k8s-master-1:~/k8s_manifests/example-podsecuritypolicy# kubectl get rs -n privileged alpine-privileged-75cf8fcc55 -o jsonpath='{.spec.template.spec.serviceAccount}'
# output
privileged-sa
```

```shell
#检查pod注释：
root@k8s-master-1:~/k8s_manifests/example-podsecuritypolicy# kubectl get pod -n privileged alpine-privileged-75cf8fcc55-cgjfk -o jsonpath='{.metadata.annotations}'
# output
map[kubernetes.io/psp:psp.privileged]
```

那么seccomp呢？在`psp.privileged`策略中没有`seccomp`定义配置文件。让我们来看看。我们要找的是`sleep 1800`：

```shell
ps ax | grep "sleep 1800"
grep Seccomp /proc/<pid of sleep 1800>/status
# output
Seccomp:	0
```

### 创建多个PodSecurityPolicies并检查策略顺序

当有多个策略可用时，pod安全策略控制器按以下顺序选择策略：

1. 如果策略成功验证pod而不更改它，则使用它们
2. 如果是pod创建请求，则使用按字母顺序排列的第一个有效策略
3. 否则，如果存在pod更新请求，则返回错误，因为在更新操作期间不允许pod突变

首先，我们将使用RBAC创建两个Pod安全策略：

*PSP-multiple.yaml：*

```yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: psp.one
  annotations:
    seccomp.security.alpha.kubernetes.io/defaultProfileName:  'runtime/default'
    seccomp.security.alpha.kubernetes.io/allowedProfileNames: 'runtime/default'
spec:
  readOnlyRootFilesystem: false
  privileged: false
  allowPrivilegeEscalation: true
  runAsUser:
    rule: 'RunAsAny'
  supplementalGroups:
    rule: 'RunAsAny'
  fsGroup:
    rule: 'RunAsAny'
  seLinux:
    rule: 'RunAsAny'
  volumes:
  - configMap
  - emptyDir
  - secret
---
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: psp.two
  annotations:
    seccomp.security.alpha.kubernetes.io/defaultProfileName:  'runtime/default'
    seccomp.security.alpha.kubernetes.io/allowedProfileNames: 'runtime/default'
spec:
  readOnlyRootFilesystem: false
  privileged: false
  allowPrivilegeEscalation: false
  runAsUser:
    rule: 'MustRunAsNonRoot'
  supplementalGroups:
    rule: 'RunAsAny'
  fsGroup:
    rule: 'RunAsAny'
  seLinux:
    rule: 'RunAsAny'
  volumes:
  - configMap
  - emptyDir
  - secret
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: psp:multy
  namespace: multy
rules:
- apiGroups:
  - policy
  resourceNames:
  - psp.one
  - psp.two
  resources:
  - podsecuritypolicies
  verbs:
  - use
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: psp:multy:binding
  namespace: multy
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: psp:multy
subjects:
  - kind: ServiceAccount
    name: replicaset-controller
    namespace: kube-system

```

检查namespace、multy-psp资源情况

```shell
root@k8s-master-1:~/k8s_manifests/example-podsecuritypolicy# kubectl apply -f multy-namespace.yaml 
namespace/multy created
root@k8s-master-1:~/k8s_manifests/example-podsecuritypolicy# kubectl apply -f PSP-multiple.yaml 
podsecuritypolicy.policy/psp.one created
podsecuritypolicy.policy/psp.two created
role.rbac.authorization.k8s.io/psp:multy created
rolebinding.rbac.authorization.k8s.io/psp:multy:binding created
root@k8s-master-1:~/k8s_manifests/example-podsecuritypolicy# kubectl get psp -n multy
NAME             PRIV    CAPS   SELINUX    RUNASUSER          FSGROUP     SUPGROUP    READONLYROOTFS   VOLUMES
psp.one          false          RunAsAny   RunAsAny           RunAsAny    RunAsAny    false            configMap,emptyDir,secret
psp.privileged   true           RunAsAny   RunAsAny           RunAsAny    RunAsAny    false            configMap,emptyDir,secret
psp.restricted   false          RunAsAny   MustRunAsNonRoot   MustRunAs   MustRunAs   true             configMap,emptyDir,secret
psp.two          false          RunAsAny   MustRunAsNonRoot   RunAsAny    RunAsAny    false            configMap,emptyDir,secret

```

`example-multy-psp-deployment.yaml`

```yanl
apiVersion: apps/v1
kind: Deployment
metadata:
  name: alpine-multy
  namespace: multy
  labels:
    app: alpine-multy
spec:
  replicas: 1
  selector:
    matchLabels:
      app: alpine-multy
  template:
    metadata:
      labels:
        app: alpine-multy
    spec:
      securityContext:
        runAsUser: 1000
        fsGroup: 2000
      containers:
      - name: alpine-multy
        image: alpine:3.9
        command: ["sleep", "2400"]
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: false
```

检查pod启用的psp策略

```shell
root@k8s-master-1:~/k8s_manifests/example-podsecuritypolicy# kubectl get pod alpine-multy-dfbf478b6-zhvlr -n multy -o jsonpath='{.metadata.annotations}'
#output
map[kubernetes.io/psp:psp.one seccomp.security.alpha.kubernetes.io/pod:runtime/default]
```

如您所见，此pod使用`psp.one`PodSecurityPolicy，它是按字母顺序排列的第一个有效策略，尽管它是PSP选择中的第二个规则:**如果是pod创建请求，则使用按字母顺序排列的第一个有效策略**。

![](https://i.loli.net/2019/06/12/5d00ba82b3bb372618.png)

删除`psp.two``的`seccomp`注释：

```yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: psp.two
spec:
  readOnlyRootFilesystem: false
  privileged: false
  allowPrivilegeEscalation: false
  runAsUser:
    rule: 'MustRunAsNonRoot'
  supplementalGroups:
    rule: 'RunAsAny'
  fsGroup:
    rule: 'RunAsAny'
  seLinux:
    rule: 'RunAsAny'
  volumes:
  - configMap
  - emptyDir
  - secret
```

运行结果

```shell
root@k8s-master-1:~/k8s_manifests/example-podsecuritypolicy# kubectl get deploy,rs,pod -n multy
NAME                                 READY   UP-TO-DATE   AVAILABLE   AGE
deployment.extensions/alpine-multy   1/1     1            1           35s

NAME                                           DESIRED   CURRENT   READY   AGE
replicaset.extensions/alpine-multy-dfbf478b6   1         1         1       35s

NAME                               READY   STATUS    RESTARTS   AGE
pod/alpine-multy-dfbf478b6-z4dc7   1/1     Running   0          34s

root@k8s-master-1:~/k8s_manifests/example-podsecuritypolicy# kubectl get pod/alpine-multy-dfbf478b6-z4dc7 -n multy -o jsonpath='{.metadata.annotations}'
#output
map[kubernetes.io/psp:psp.two]

```

现在，pod `psp.two`由于删除了注释，因为在这种情况下，策略成功验证了pod而不更改它; 这是PSP选择的第一条规则: **如果任何策略成功验证了pod而不更改它，则使用它们**

![](https://i.loli.net/2019/06/12/5d00bd16bddab26944.png)

## 结论

正如这三个示例所示，PSP使管理员能够对Kubernetes中运行的pod和容器应用细粒度控制，方法是授予或拒绝对特定资源的访问。这些策略相对容易创建和部署，应该是任何Kubernetes安全策略的有用组件。
