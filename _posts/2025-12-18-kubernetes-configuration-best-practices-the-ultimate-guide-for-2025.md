---
layout: mypost
title: Kubernetes Configuration Best Practices The Ultimate Guide for 2025
categories: [Kubernetes]  
---

# Kubernetes 配置最佳实践：2025 终极指南

## 引言：为什么配置在 Kubernetes 中如此重要

配置是每个 Kubernetes 部署的基石。一个错位的 YAML 缩进、一个不正确的 API 版本，或者一个缺失的标签选择器都可能导致整个应用崩溃。根据最近的 DevOps 调查，**超过 60% 的 Kubernetes 事故是由配置错误引起的**。

这份综合指南汇集了经过生产环境验证的 Kubernetes 配置最佳实践。无论你是部署第一个容器化应用的初学者，还是管理企业集群的资深平台工程师，这些实践都将帮助你构建更可靠、更易维护、更安全的 Kubernetes 部署。

**你将学到：**

- 如何编写清晰、易维护的 Kubernetes YAML 文件
- Pod、Deployment、Service 和其他工作负载的最佳实践
- 跨环境管理配置的策略
- 通过正确配置加强安全性
- 节省时间的 kubectl 命令和工具

---

## 通用 Kubernetes 配置最佳实践

### 1. 始终使用最新的稳定 API 版本

Kubernetes API 演进迅速。使用已弃用的 API 版本会在升级集群时导致部署失败。

**始终检查可用的 API 版本：**

```bash
kubectl api-resources
kubectl api-versions
```

**专业提示：** 使用 `kubent`（Kube No Trouble）等工具在升级前检测清单中已弃用的 API：

```bash
kubent --helm3 --exit-error
```

### 2. 将所有配置存储在版本控制中

**永远不要直接从本地机器应用清单。** 每个 Kubernetes 配置文件都应该存放在 Git（或其他版本控制系统）中。

**版本控制配置的好处：**

- 部署失败时可即时回滚
- 完整的审计跟踪，记录谁在何时更改了什么
- 便于协作和代码审查流程
- 跨环境可重现的集群设置

**推荐的文件夹结构：**

```
kubernetes/
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── configmap.yaml
├── overlays/
│   ├── development/
│   ├── staging/
│   └── production/
└── README.md
```

### 3. 使用 Kustomize 或 Helm 保持配置 DRY

不要在不同环境中重复自己。使用 **Kustomize**（内置于 kubectl）或 **Helm** 来管理特定环境的变化。

**Kustomize 示例：**

```yaml
# kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
patches:
  - path: replica-patch.yaml
```

### 4. 使用 Namespace 进行逻辑隔离

Namespace 在集群内提供逻辑边界。使用它们来：

- 分离环境（dev、staging、prod）
- 隔离团队或项目
- 应用资源配额和网络策略

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    environment: production
    team: platform
```

---

## YAML 配置标准

### 5. 使用 YAML 而非 JSON 编写配置

虽然 Kubernetes 同时接受 YAML 和 JSON，但 **YAML 是社区标准**。它更易读，支持注释，更容易维护。

**YAML 最佳实践：**

```yaml
# 好的做法：清晰、可读的 YAML
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-application
  labels:
    app.kubernetes.io/name: web-app
    app.kubernetes.io/component: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      app.kubernetes.io/name: web-app
```

### 6. 正确处理 YAML 布尔值

YAML 的布尔值解析比较棘手。不同的 YAML 版本对值的解释不同。

**始终使用明确的布尔值：**

```yaml
# ✅ 正确 - 始终使用 true/false
enabled: true
secure: false

# ❌ 避免 - 这些可能导致问题
enabled: yes    # 可能无法正确解析
enabled: on     # 含义不明确
enabled: "yes"  # 这是字符串，不是布尔值
```

### 7. 保持清单简洁

不要设置 Kubernetes 默认处理的值。简洁的清单：

- 更易于阅读和审查
- 更不容易出错
- 更易于维护

**简洁 Deployment 示例：**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
```

### 8. 在单个文件中组合相关对象

如果你的 Deployment、Service 和 ConfigMap 属于同一个应用，将它们放在一个文件中，用 `---` 分隔。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  # ... deployment spec
---
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  # ... service spec
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-config
data:
  # ... configuration data
```

**好处：**

- 原子部署（全有或全无）
- 更容易管理相关资源
- 在版本控制中更好的组织

---

## Pod 和工作负载配置最佳实践

### 9. 生产环境中永远不要使用裸 Pod

**裸 Pod**（不由控制器管理的 Pod）在生产环境中很危险，因为：

- 节点故障时它们不会重新调度
- 它们不会自动扩展
- 它们缺乏滚动更新能力

**始终使用控制器：**

| 使用场景 | 控制器 |
|----------|--------|
| 长期运行的应用 | Deployment |
| 有状态应用 | StatefulSet |
| 节点级守护进程 | DaemonSet |
| 批处理 | Job |
| 定时任务 | CronJob |

### 10. 对无状态应用使用 Deployment

Deployment 是运行无状态应用的标准：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
  annotations:
    kubernetes.io/description: "处理客户数据的 REST API 服务器"
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: api-server
  template:
    metadata:
      labels:
        app: api-server
        version: v2.1.0
    spec:
      containers:
      - name: api
        image: myregistry/api-server:v2.1.0
        ports:
        - containerPort: 8080
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
```

### 11. 对一次性任务使用 Job

Job 非常适合：

- 数据库迁移
- 批处理
- 数据导入/导出

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
spec:
  backoffLimit: 3
  ttlSecondsAfterFinished: 3600
  template:
    spec:
      restartPolicy: OnFailure
      containers:
      - name: migration
        image: myregistry/db-migrate:v1.0
        command: ["./migrate", "--target=latest"]
```

### 12. 始终配置健康检查

健康检查对生产环境的可靠性至关重要：

- **存活探针（Liveness Probe）：** 重启卡住的容器
- **就绪探针（Readiness Probe）：** 在 Pod 就绪前将其从服务端点移除
- **启动探针（Startup Probe）：** 处理启动缓慢的容器

```yaml
containers:
- name: app
  image: myapp:v1
  livenessProbe:
    httpGet:
      path: /healthz
      port: 8080
    initialDelaySeconds: 30
    periodSeconds: 10
    failureThreshold: 3
  readinessProbe:
    httpGet:
      path: /ready
      port: 8080
    initialDelaySeconds: 5
    periodSeconds: 5
    successThreshold: 1
  startupProbe:
    httpGet:
      path: /healthz
      port: 8080
    failureThreshold: 30
    periodSeconds: 10
```

---

## Deployment 配置策略

### 13. 正确配置滚动更新

滚动更新通过逐步替换旧 Pod 来最小化停机时间：

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%        # 更新期间最多额外的 Pod 数
      maxUnavailable: 0    # 零停机要求
```

**关键设置说明：**

- `maxSurge`：发布期间可以存在多少额外的 Pod
- `maxUnavailable`：发布期间可以有多少 Pod 不可用

**零停机部署配置：**

```yaml
maxSurge: 1
maxUnavailable: 0
```

### 14. 使用 Pod 中断预算（PDB）

PDB 在自愿中断（节点排空、集群升级）期间保护你的应用：

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: api-pdb
spec:
  minAvailable: 2    # 或使用 maxUnavailable
  selector:
    matchLabels:
      app: api-server
```

### 15. 实施 Pod 反亲和性以实现高可用

将 Pod 分散到节点和可用区以应对故障：

```yaml
spec:
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app: api-server
          topologyKey: kubernetes.io/hostname
      - weight: 50
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app: api-server
          topologyKey: topology.kubernetes.io/zone
```

---

## Service 和网络配置

### 16. 在依赖的工作负载之前创建 Service

Kubernetes 在启动时将 Service 环境变量注入 Pod。首先创建 Service 以确保环境变量可用：

```bash
# 按此顺序部署：
kubectl apply -f configmap.yaml
kubectl apply -f service.yaml
kubectl apply -f deployment.yaml
```

**环境变量格式：**

```bash
# 对于名为 "database" 的 Service
DATABASE_SERVICE_HOST=10.96.0.10
DATABASE_SERVICE_PORT=5432
```

### 17. 使用 DNS 进行服务发现

DNS 比环境变量更灵活：

```bash
# 通过 DNS 访问服务
http://my-service.my-namespace.svc.cluster.local
http://my-service.my-namespace  # 简短形式
http://my-service              # 在同一 namespace 内
```

### 18. 避免使用 hostPort 和 hostNetwork

这些选项限制调度并带来安全风险：

```yaml
# ❌ 除非绝对必要，否则避免使用
spec:
  hostNetwork: true
  containers:
  - name: app
    ports:
    - containerPort: 80
      hostPort: 80  # 将 Pod 绑定到特定节点
```

**更好的替代方案：**

- 使用 `NodePort` Service 进行外部访问
- 在云环境中使用 `LoadBalancer` Service
- 对 HTTP 流量使用 Ingress 控制器
- 使用 `kubectl port-forward` 进行调试

### 19. 对 StatefulSet 发现使用 Headless Service

用于直接的 Pod 到 Pod 通信：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  clusterIP: None  # Headless service
  selector:
    app: mysql
  ports:
  - port: 3306
```

这将启用如下 DNS 记录：

```
mysql-0.mysql.default.svc.cluster.local
mysql-1.mysql.default.svc.cluster.local
```

---

## 标签、选择器和注解

### 20. 一致地使用语义化标签

标签是连接 Kubernetes 资源的粘合剂：

```yaml
metadata:
  labels:
    # 推荐的 Kubernetes 标签
    app.kubernetes.io/name: web-app
    app.kubernetes.io/instance: web-app-prod
    app.kubernetes.io/version: "2.1.0"
    app.kubernetes.io/component: frontend
    app.kubernetes.io/part-of: ecommerce
    app.kubernetes.io/managed-by: helm
    
    # 自定义组织标签
    team: platform
    environment: production
    cost-center: cc-123
```

### 21. 标签选择策略

使用标签进行强大的资源选择：

```bash
# 获取所有前端 Pod
kubectl get pods -l app.kubernetes.io/component=frontend

# 获取所有生产资源
kubectl get all -l environment=production

# 删除所有测试资源
kubectl delete all -l environment=test

# 组合选择器
kubectl get pods -l 'app=web,environment in (staging,production)'
```

### 22. 使用注解存储元数据

注解存储非标识信息：

```yaml
metadata:
  annotations:
    kubernetes.io/description: "处理客户请求的主 API 服务器"
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080"
    deployment.kubernetes.io/revision: "3"
    kubectl.kubernetes.io/last-applied-configuration: |
      {...}
```

### 23. 使用标签操作进行调试

临时从服务中移除 Pod 以进行调试：

```bash
# 隔离 Pod（从服务端点移除）
kubectl label pod myapp-pod-xyz app-

# Pod 继续运行但不接收流量
# 调试 Pod
kubectl exec -it myapp-pod-xyz -- /bin/sh

# 完成后，删除隔离的 Pod
kubectl delete pod myapp-pod-xyz
```

---

## ConfigMap 和 Secret 管理

### 24. 对非敏感配置使用 ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  # 简单的键值对
  LOG_LEVEL: "info"
  MAX_CONNECTIONS: "100"
  
  # 完整的配置文件
  config.yaml: |
    server:
      port: 8080
      timeout: 30s
    features:
      enable_cache: true
```

**在 Pod 中使用：**

```yaml
spec:
  containers:
  - name: app
    envFrom:
    - configMapRef:
        name: app-config
    volumeMounts:
    - name: config
      mountPath: /etc/config
  volumes:
  - name: config
    configMap:
      name: app-config
```

### 25. 安全地处理 Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
stringData:  # 使用 stringData 输入明文
  username: admin
  password: supersecret123
```

**安全建议：**

- 为 etcd 启用静态加密
- 使用外部密钥管理器（Vault、AWS Secrets Manager）
- 为 Secret 访问实施 RBAC
- 定期轮换密钥
- 永远不要将密钥提交到版本控制

---

## 资源管理和限制

### 26. 始终设置资源请求和限制

资源配置可防止"吵闹的邻居"并确保调度：

```yaml
spec:
  containers:
  - name: app
    resources:
      requests:
        memory: "256Mi"
        cpu: "100m"
      limits:
        memory: "512Mi"
        cpu: "500m"
```

**指南：**

- `requests`：保证的资源（用于调度）
- `limits`：允许的最大资源
- 将 `requests` 设置为接近实际使用量
- 将内存 `limit` = `request` 以防止 OOMKilled 问题
- CPU 限制可以是请求的 2-4 倍以提供突发容量

### 27. 对 Namespace 限制使用 ResourceQuota

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-quota
  namespace: team-alpha
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
    pods: "50"
```

### 28. 实施 LimitRange 设置默认值

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
spec:
  limits:
  - type: Container
    default:
      memory: "256Mi"
      cpu: "200m"
    defaultRequest:
      memory: "128Mi"
      cpu: "100m"
```

---

## 安全配置最佳实践

### 29. 以非 root 用户运行容器

```yaml
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 1000
    fsGroup: 1000
  containers:
  - name: app
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
          - ALL
```

### 30. 使用网络策略

限制 Pod 到 Pod 的通信：

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-network-policy
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: database
    ports:
    - port: 5432
```

### 31. 使用 Pod 安全标准

在 Namespace 级别应用安全标准：

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/audit: restricted
```

---

## kubectl 配置管理技巧

### 32. 应用整个目录

```bash
# 应用目录中的所有清单
kubectl apply -f ./kubernetes/ --recursive

# 使用服务端应用以更好地处理冲突
kubectl apply -f ./kubernetes/ --server-side

# 应用前预览更改
kubectl diff -f ./kubernetes/
```

### 33. 从运行的资源生成清单

```bash
# 将运行的 Deployment 导出为 YAML
kubectl get deployment nginx -o yaml > nginx-deployment.yaml

# 命令式创建 Deployment，然后导出
kubectl create deployment nginx --image=nginx --dry-run=client -o yaml > deployment.yaml
```

### 34. 应用前验证配置

```bash
# 干运行验证
kubectl apply -f deployment.yaml --dry-run=server

# 客户端验证
kubectl apply -f deployment.yaml --dry-run=client

# 使用 kubeval 验证
kubeval deployment.yaml

# 使用 kubeconform 验证（更快）
kubeconform -strict deployment.yaml
```

---

## 要避免的常见配置错误

| 错误 | 影响 | 解决方案 |
|------|------|----------|
| 没有资源限制 | 节点不稳定、OOM 终止 | 始终设置请求和限制 |
| 缺少健康检查 | 未检测到的故障、恢复不佳 | 配置存活和就绪探针 |
| 裸 Pod | 没有自动恢复 | 使用 Deployment 或其他控制器 |
| 硬编码密钥 | 安全漏洞 | 使用 Kubernetes Secret 或外部保管库 |
| 没有 PodDisruptionBudget | 维护期间停机 | 为关键工作负载创建 PDB |
| 不正确的标签选择器 | 孤立资源、路由问题 | 精确匹配标签 |
| 以 root 运行 | 安全风险 | 使用 runAsNonRoot: true |
| 没有资源请求 | 调度决策不佳 | 设置实际的请求 |

---

## 配置验证工具

### 推荐工具

1. **kubeval / kubeconform**：根据 Kubernetes schema 验证 YAML
2. **kube-linter**：最佳实践的静态分析
3. **Datree**：策略执行和验证
4. **OPA/Gatekeeper**：策略即代码验证
5. **Polaris**：最佳实践和安全检查
6. **kubent**：检测已弃用的 API

**CI/CD 验证管道示例：**

```yaml
# .github/workflows/validate.yaml
name: Validate Kubernetes Manifests
on: [pull_request]
jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - name: Validate schemas
      run: kubeconform -strict -summary kubernetes/
    - name: Check best practices
      run: kube-linter lint kubernetes/
    - name: Detect deprecated APIs
      run: kubent -f kubernetes/
```

---

## 结论

Kubernetes 配置可能看起来平淡无奇，但它是可靠部署的基础。通过遵循这些最佳实践，你将构建出以下特点的集群：

- **可维护**：清晰、版本控制的配置
- **可靠**：适当的健康检查、资源限制和反亲和性规则
- **安全**：非 root 容器、网络策略和密钥管理
- **可扩展**：正确使用控制器和资源管理

**关键要点：**

1. 始终对配置进行版本控制
2. 使用控制器（Deployment、StatefulSet）而不是裸 Pod
3. 为每个容器设置资源请求和限制
4. 为所有工作负载配置健康检查
5. 使用一致的语义化标签
6. 应用前验证配置

今天就开始实施这些实践，当凌晨 3 点调试成为罕见情况时，你未来的自己（和你的团队）会感谢你。

---

**相关文章：**

- Kubernetes 安全最佳实践
- Docker 到 Kubernetes 迁移指南
- Helm Chart 完整教程
