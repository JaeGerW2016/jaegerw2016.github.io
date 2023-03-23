---
title: 基于RBAC创建具有有限命名空间访问权限的用户
date: 2019-07-18 14:23
tag: kubernetes
---

### 介绍

本指南将介绍基本的Kubernetes基于角色的访问控制（RBAC）API对象，以及两个常见用例（创建具有有限访问权限的用户，并启用Helm）。您应该有足够的知识在集群中实施RBAC策略

从Kubernetes 1.6开始，默认情况下启用RBAC策略。RBAC策略对于正确管理群集至关重要，因为它们允许您根据用户及其在组织中的角色指定允许的操作类型。例子包括：

- 通过仅向管理员用户授予特权操作（例如访问secrets）来保护您的集群。
- 强制群集中的用户身份验证。
- 将资源创建（例如pod，pv，deployment）限制为特定namespace。您还可以使用配额来确保资源使用受到限制并受到控制。
- 让用户只在其授权的命名空间中查看资源。这允许您隔离组织内的资源（例如，在部门之间）。

由于默认情况下启用了RBAC，因此在配置网络覆盖（例如flanneld）或使Helm在群集中工作时，您可能会发现这样的错误：

```
the server does not allow access to the requested resource
```

本文将向您展示如何使用RBAC，以便您可以正确处理这些问题。

### 先决条件和假设

本指南做出以下假设：

- 您正在运行[Kubernetes集群](https://docs.bitnami.com/kubernetes/get-started-kubernetes/#option-1-create-a-cluster-using-minikube)。
- 您已安装[*kubectl*命令行（kubectl CLI）](https://docs.bitnami.com/kubernetes/get-started-kubernetes/#step-3-install-kubectl-command-line)。
- 你安装了[Helm和Tiller](https://docs.bitnami.com/kubernetes/get-started-kubernetes/#step-4-install-helm-and-tiller)。
- 您对[Kubernetes的工作方式](https://kubernetes.io/docs/concepts/)及其[核心资源和运营](https://kubernetes.io/docs/concepts/overview/kubernetes-api/)有中等程度的了解。您应该熟悉以下概念：
  - [Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod/)
  - [deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
  - [namespace](https://kubernetes.io/docs/user-guide/namespaces/)
  - [secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
  - [Replicasets](https://kubernetes.io/docs/user-guide/replicasets/)
  - [PersistentVolumes](https://kubernetes.io/docs/concepts/storage/volumes/)
  - [ConfigMaps](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/)
  - [node](https://kubernetes.io/docs/concepts/architecture/nodes/)
- 您已在本地安装[OpenSSL](https://wiki.openssl.org/index.php/Binaries)

### RBAC API对象

一个基本的Kubernetes功能是它的[所有资源都是建模的API对象](https://kubernetes.io/docs/concepts/overview/working-with-objects/kubernetes-objects/)，它允许CRUD（创建，读取，更新，删除）操作。资源的例子是：

- [Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod/)
- [PersistentVolumes](https://kubernetes.io/docs/concepts/storage/volumes/)
- [ConfigMaps](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/)
- [deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [node](https://kubernetes.io/docs/concepts/architecture/nodes/)
- [secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
- [namespace](https://kubernetes.io/docs/user-guide/namespaces/)

这些资源的可能操作示例如下：

- *create*
- *get*
- *delete*
- *list*
- *update*
- *edit*
- *watch*
- *exec*

在更高级别，资源与[API组](https://kubernetes.io/docs/concepts/overview/kubernetes-api/#api-groups)相关联（例如，Pod属于*核心* API组，而Deployments属于*apps* API组）。有关所有可用资源，操作和API组的更多信息，请查看[官方Kubernetes API参考](https://kubernetes.io/docs/reference/kubernetes-api/)。

要管理Kubernetes中的RBAC，除资源和运营外，我们还需要以下要素：

- 规则：规则是一组操作（动词），可以在属于不同API组的一组资源上执行。
- 角色和ClusterRoles：两者都包含规则。Role和ClusterRole之间的区别在于范围：在Role中，规则适用于单个命名空间，而ClusterRole是群集范围的，因此规则适用于多个命名空间。ClusterRoles也可以为集群范围的资源（例如节点）定义规则。Roles和ClusterRoles都在我们的集群中映射为API资源。
- 主题：这些对应于尝试在群集中进行操作的实体。有三种类型的科目：
  - 用户帐户：这些是全局的，适用于居住在群集之外的人或流程。Kubernetes集群中没有关联的资源API对象。
  - 服务帐户：此类帐户是命名空间，适用于在pod中运行的集群内进程，这些进程需要针对API进行身份验证。
  - 组：用于引用多个帐户。默认情况下会创建一些组，例如*cluster-admin*（在后面的部分中进行了解释）。
- RoleBindings和ClusterRoleBindings：正如名称所表达意思的那样，这些将主体绑定到角色（即给定用户可以执行的操作）。
- 对于Roles和ClusterRoles，区别在于范围：RoleBinding将使规则在命名空间内有效，而ClusterRoleBinding将使规则在所有命名空间中有效。

您可以在[Kubernetes官方文档中](https://kubernetes.io/docs/admin/authorization/rbac/)找到每个API元素的[示例](https://kubernetes.io/docs/admin/authorization/rbac/)。

### 示例1：创建具有有限命名空间访问权限的用户

在此示例中，我们将创建以下用户帐户：

- user：employee
- group：lab

我们将添加必要的RBAC策略，以便此用户可以仅在*office*命名空间内完全管理部署（即使用*kubectl run*命令）。最后，我们将测试策略以确保它们按预期工作。



#### 第1步：创建Office命名空间

- 执行*kubectl create*命令创建命名空间（作为admin用户）：

  ```
  kubectl create namespace office
  ```

#### 第2步：创建用户凭据

如前所述，Kubernetes没有用户帐户的API对象。在管理身份验证的可用方法中（有关完整列表，请参阅[Kubernetes官方文档](https://kubernetes.io/docs/admin/authentication)），我们将使用OpenSSL证书来简化它们。必要的步骤是：

- 为您的用户创建私钥。在此示例中，我们将文件命名为*employee.key*：

  ```
  openssl genrsa -out employee.key 2048
  ```

- 使用刚创建的私钥（本例中为*employee.key*）创建证书签名请求*employee.csr*。确保在*-subj*部分中指定了用户名和组（CN用于用户名，O用于组）。如前所述，我们将使用*employee*作为名称，*lab*作为组：

  ```
  openssl req -new -key employee.key -out employee.csr -subj "/CN=employee/O=lab"
  ```

- 找到您的Kubernetes集群证书颁发机构（CA）。这将负责批准请求并生成访问群集API所需的证书。它的位置通常是*/ etc / kubernetes / ssl/*。在Minikube的情况下，它将是*〜/ .minikube /*。检查位置中是否存在*ca.crt*和*ca.key*文件。

- 产生最终的证书*employee.crt*通过批准证书标志的要求，*employee.csr*，你提前进行。确保将CA_LOCATION占位符替换为群集CA的位置。在此示例中，证书有效期为10年：

  ```shell
  openssl x509 -req -in employee.csr -CA /etc/kubernetes/ssl/ca.pem -CAkey /etc/kubernetes/ssl/ca-key.pem -CAcreateserial -out employee.crt -days 3650
  ```

- 将*employee.crt*和*employee.key*保存在安全的位置（在本例中我们将使用*/opt/rbac-employee/*）。

- 使用Kubernetes集群的新凭据添加新上下文。

  ```shell
  root@k8s-master-1:~# kubectl config set-credentials employee --client-certificate=/opt/rbac-employee/employee.crt  --client-key=/opt/rbac-employee/employee.key
  User "employee" set.
  root@k8s-master-1:~# kubectl config set-context employee-context --cluster=kubernetes --namespace=office --user=employee
Context "employee-context" created.
  ```

  现在，在将*kubectl* CLI与此配置文件一起使用时，您应该获得访问被拒绝错误。这是预期的，因为我们尚未为此用户定义任何允许的操作。
  
  ```shell
  root@k8s-master-1:~# kubectl --context=employee-context get pods
  error: You must be logged in to the server (Unauthorized)
  ```



#### 第3步：创建管理部署的角色

- 使用以下内容创建*deployment-manager-role.yaml*文件。在这个*yaml*文件中，我们创建了一个规则，允许用户在Deployments，Pods和ReplicaSets（创建部署所必需的）上执行多个操作，这些操作属于*核心*（由*yaml*文件中的“”表示），*apps*和*扩展* API组：

  ```yaml
  kind: Role
  apiVersion: rbac.authorization.k8s.io/v1beta1
  metadata:
    namespace: office
    name: deployment-manager
  rules:
  - apiGroups: ["", "extensions", "apps"]
    resources: ["deployments", "replicasets", "pods"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"] # You can also use ["*"]
  ```

- 使用*kubectl create role*命令在集群中*创建角色*：

  ```shell
  kubectl create -f deployment-manager-role.yaml
  ```



#### 第4步：将角色绑定到员工用户

- 使用以下内容创建*deployment-manager-rolebinding.yaml*文件。在此文件中，我们将*Deployment-manager*角色绑定到*office*命名空间内的用户帐户*员工*：

  ```yaml
  kind: RoleBinding
  apiVersion: rbac.authorization.k8s.io/v1beta1
  metadata:
    name: deployment-manager-binding
    namespace: office
  subjects:
  - kind: User
    name: employee
    apiGroup: rbac.authorization.k8s.io
  roleRef:
    kind: Role
    name: deployment-manager
    apiGroup: rbac.authorization.k8s.io
  ```

- 通过运行*kubectl create*命令来部署RoleBinding ：

  ```shell
  kubectl create -f deployment-manager-rolebinding.yaml
  ```



#### 第5步：测试RBAC规则

现在您应该能够执行以下命令而不会出现任何问题：

```shell
root@k8s-master-1:/opt/rbac-employee# kubectl --context=employee-context run --image bitnami/dokuwiki mydokuwiki
kubectl run --generator=deployment/apps.v1 is DEPRECATED and will be removed in a future version. Use kubectl run --generator=run-pod/v1 or kubectl create instead.
deployment.apps/mydokuwiki created
root@k8s-master-1:/opt/rbac-employee# kubectl --context=employee-context get pods
NAME                          READY   STATUS              RESTARTS   AGE
mydokuwiki-77c588b665-btdwc   0/1     ContainerCreating   0          9s
```

如果使用*--namespace = default*参数运行相同的命令，则它将失败，因为*employee*用户无权访问此命名空间。

```shell
root@k8s-master-1:/opt/rbac-employee# kubectl config get-contexts 
CURRENT   NAME               CLUSTER      AUTHINFO   NAMESPACE
          employee-context   kubernetes   employee   office
*         kubernetes         kubernetes   admin
```

```shell
root@k8s-master-1:/opt/rbac-employee# kubectl --context=employee-context get pods --namespace=default
Error from server (Forbidden): pods is forbidden: User "employee" cannot list resource "pods" in API group "" in the namespace "default"
```

现在，您已在群集中创建了权限有限的用户。
#### 第6步：生成普通用户token并赋予特定namespace的admin权限 登录kubedashboard

为指定namespace分配该namespace的最高权限，这通常是在为某个用户（组织或者个人）划分了namespace之后，需要给该用户创建token登陆kubernetes dashboard或者调用kubernetes API的时候使用。

每次创建了新的namespace下都会生成一个默认的token，名为`default-token-xxxx`。`default`就相当于该namespace下的一个用户，可以使用下面的命令生成一个普通用户employee分配该namespace的管理员权限。

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: employee
  namespace: office
```

```shell
root@k8s-master-1:/opt/rbac-employee# kubectl get secrets  -n office
NAME                   TYPE                                  DATA   AGE
default-token-dcf52    kubernetes.io/service-account-token   3      66m
employee-token-lrm47   kubernetes.io/service-account-token   3      33s
```

```shell
root@k8s-master-1:/opt/rbac-employee# kubectl create rolebinding employee --clusterrole=admin --serviceaccount=office:employee --namespace=office
rolebinding.rbac.authorization.k8s.io/employee created
```

```shell
root@k8s-master-1:/opt/rbac-employee# kubectl describe secrets -n office employee-token-lrm47 
Name:         employee-token-lrm47
Namespace:    office
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: employee
              kubernetes.io/service-account.uid: 09c08afc-a927-11e9-acaa-000c2954713e

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1346 bytes
namespace:  6 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJvZmZpY2UiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlY3JldC5uYW1lIjoiZW1wbG95ZWUtdG9rZW4tbHJtNDciLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoiZW1wbG95ZWUiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiIwOWMwOGFmYy1hOTI3LTExZTktYWNhYS0wMDBjMjk1NDcxM2UiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6b2ZmaWNlOmVtcGxveWVlIn0.q-rMg1va-jNKuTQ9btllox_GRKSKPG1VqrRCDmEJoAfZPsiBxsC3AsQeDykivqH8Vm8WAdncGic7fvltnPJLy_siBjATtahyqebW6-6Hna2S6PY1Ljn_miSwODUFxsUfZUodK9FpMv0qKI4gHZVD-BjlQ1aFgw3HyAYdnfES_Q_ulhKHVjgRBCSYfUYe8Vj4tpuqlGRJbevUy6LQaQzcoUeUFdJOaZHgoQO4R9J1yCwVtwQoAuq2vyoLA03oKN19dqUe7dXfyMUTR1aSyrrBUzJ_1pEbZKesXxSSgZ84D0N7Q7G8CnExNZ00xvm-Di1tLjQsb1DdEX9YxLexto_5Sx
```

![](https://i.loli.net/2019/07/18/5d30164ae977371306.png)

