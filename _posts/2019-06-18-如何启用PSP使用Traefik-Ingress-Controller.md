---
title: 如何启用PSP使用Traefik Ingress Controller
date: 2019-06-18 08:54:00
tags: kubernetes
---

如何启用PSP使用Traefik Ingress Controller

#### 集群环境

```shell
root@k8s-master-1:~# kubectl get nodes -o wide
NAME           STATUS                     ROLES       AGE    VERSION   INTERNAL-IP    EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
192.168.2.10   Ready,SchedulingDisabled   master      114d   v1.13.3   192.168.2.10   <none>        Ubuntu 18.04 LTS     4.15.0-51-generic   docker://18.9.2
192.168.2.11   Ready                      node        114d   v1.13.3   192.168.2.11   <none>        Ubuntu 18.04 LTS     4.15.0-51-generic   docker://18.9.2
192.168.2.12   Ready                      node        114d   v1.13.3   192.168.2.12   <none>        Ubuntu 18.04.2 LTS   4.15.0-51-generic   docker://18.9.2
192.168.2.13   Ready                      edge,node   114d   v1.13.3   192.168.2.13   <none>        Ubuntu 18.04 LTS     4.15.0-51-generic   docker://18.9.2

```

#### 授权Traefik使用您的Kubernetes集群

kubernetes版本使用[RBAC（基于角色的访问控制）](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)，允许Kubernetes资源以**ServiceAccount**方式与其API通信。有两种方法可以设置权限以允许Traefik资源与k8s API进行通信：

- 通过RoleBinding（特定于命名空间）
- 通过ClusterRoleBinding（所有命名空间的全局）

为简单起见，我们将使用`ClusterRoleBinding`以便在集群级别和所有名称空间中授予权限。以下内容`ClusterRoleBinding`允许属于其中的任何用户或资源`ServiceAccount`使用traefik入口控制器。

`traefik-rbac.yaml`

```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: traefik-ingress-controller
  namespace: kube-system
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: traefik-ingress-controller
rules:
  - apiGroups:
      - extensions
    resources:  
      - podsecuritypolicies
    verbs: 
      - use
    resourceNames: 
      - psp.traefik.unprivileged
  - apiGroups:
      - ""
    resources:
      - pods
      - services
      - endpoints
      - secrets
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - extensions
    resources:
      - ingresses
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - extensions
    resources:
      - ingresses/status
    verbs:
      - update
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: traefik-ingress-controller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: traefik-ingress-controller
subjects:
- kind: ServiceAccount
  name: traefik-ingress-controller
  namespace: kube-system
```

```shell
$ kubectl apply -f traefik-rbac.yaml
serviceaccount/traefik-ingress-controller created
clusterrole.rbac.authorization.k8s.io/traefik-ingress-controller created
clusterrolebinding.rbac.authorization.k8s.io/traefik-ingress-controller created
```

#### 启用PodSecurityPolicy

`traefik-ds-psp.yaml`

```yaml
apiVersion: extensions/v1beta1
kind: PodSecurityPolicy
metadata:
  name: psp.traefik.unprivileged
spec:
  privileged: true
  volumes:
    - configMap
    - secret
    - hostPath
  readOnlyRootFilesystem: false
  runAsUser:
    rule: RunAsAny
  supplementalGroups:
    rule: RunAsAny
  fsGroup:
    rule: RunAsAny
  allowPrivilegeEscalation: true
  defaultAllowPrivilegeEscalation: true
  allowedCapabilities: ['NET_BIND_SERVICE']
  defaultAddCapabilities: []
  requiredDropCapabilities: []
  hostPID: true
  hostIPC: true
  hostNetwork: false
  hostPorts:
  - max: 65535
    min: 1
  seLinux:
    rule: 'RunAsAny'
```

```shell
$ kubectl apply -f traefik-ds-psp.yaml

podsecuritypolicy.extensions/psp.traefik.unprivileged created
```

您可以通过键入来验证是否已加载：

```
$ kubectl get psp
NAME                                           PRIV    CAPS               SELINUX    RUNASUSER          FSGROUP     SUPGROUP    READONLYROOTFS   VOLUMES
psp.traefik.unprivileged                       true    NET_BIND_SERVICE   RunAsAny   RunAsAny           RunAsAny    RunAsAny    false            configMap,secret,hostPath

```

#### 部署Daemonset Traefik Ingress Controller

`traefik-ds.yaml`

```yaml
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: traefik-ingress-controller
  namespace: kube-system
  labels:
    k8s-app: traefik-ingress-lb
spec:
  template:
    metadata:
      labels:
        k8s-app: traefik-ingress-lb
        name: traefik-ingress-lb
    spec:
      nodeSelector:
        traefik: "svc"
      serviceAccountName: traefik-ingress-controller
      terminationGracePeriodSeconds: 60
      volumes:
      - name: ssl
        secret:
          secretName: traefik-cert
      - name: config
        configMap:
          name: traefik-conf
      containers:
      - image: traefik
        name: traefik-ingress-lb
        volumeMounts:
        - mountPath: "/ssl"
          name: ssl
        - mountPath: "/config"
          name: config
        ports:
        - name: http
          containerPort: 80
          hostPort: 80
        - name: https
          containerPort: 443
          hostPort: 443
        - name: admin
          containerPort: 8080
          hostPort: 8080
        securityContext:
          capabilities:
            drop:
            - ALL
            add:
            - NET_BIND_SERVICE
        args:
        - --configfile=/config/traefik.toml
        - --api
        - --kubernetes
        - --logLevel=ERROR
      initContainers:
      - image: busybox
        command:
        - sh
        - -c
        - echo 65535 > /proc/sys/net/core/somaxconn
        imagePullPolicy: IfNotPresent
        name: setsysctl
        securityContext:
          privileged: true    
```

#### 结果验证

```
$ kubectl get ds -n kube-system
NAME                         DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR                   AGE
traefik-ingress-controller   1         1         1       1            1           traefik=svc                     5h6m

$ kubectl get svc -n kube-system
traefik-ingress-service                       ClusterIP   10.68.252.76    <none>        80/TCP,8080/TCP,443/TCP   114d

$ kubectl get pod -n kube-system
NAME                                      READY   STATUS    RESTARTS   AGE
traefik-ingress-controller-ghghg          1/1     Running   0          5h15m

```




