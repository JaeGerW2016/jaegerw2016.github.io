---
layout: mypost
title: 在Kubernetes环境下本地部署FastGPT完整指南
categories: [kubernetes,devops]
---

### 背景

FastGPT 是一个基于 LLM（大语言模型）的知识库问答系统，提供了开箱即用的知识库、多模态文件解析、工作流编排等功能。它支持多种大模型接入，可以快速构建企业级的 AI 知识库应用。

在 Kubernetes 环境中部署 FastGPT 可以充分利用容器编排的优势，实现高可用、易扩展、易维护的部署方案。本文将详细介绍如何在 Kubernetes 环境中部署完整的 FastGPT 系统。

### FastGPT 架构概述

FastGPT 是一个微服务架构的系统，包含以下核心组件：

1. **fastgpt-app** - 主应用服务，提供 Web 界面和 API
2. **fastgpt-mongo** - MongoDB 数据库，存储应用数据
3. **fastgpt-redis** - Redis 缓存，提供缓存和会话管理
4. **fastgpt-minio** - MinIO 对象存储，存储文件和媒体资源
5. **fastgpt-pg** - PostgreSQL 数据库（带 pgvector 扩展），存储向量数据
6. **fastgpt-aiproxy** - AI 代理服务，统一管理 AI 模型调用
7. **fastgpt-code-sandbox** - 代码沙箱，安全执行 JavaScript 和 Python 代码
8. **fastgpt-mcp-server** - MCP 服务器，提供模型上下文协议服务
9. **fastgpt-plugin** - 插件服务，支持插件扩展
10. **fastgpt-volume-manager** - 卷管理器，管理 Docker 卷

### 准备条件

- **Kubernetes Cluster**（建议 v1.20+）
- **kubectl** 命令行工具
- **Helm**（可选，用于包管理）
- **足够的存储资源**（建议至少 5GB 可用空间）
- **网络访问权限**（用于拉取镜像）

### 部署步骤

#### 1. 创建命名空间

首先创建一个独立的命名空间来部署 FastGPT：

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: fastgpt
```

应用配置：

```bash
kubectl apply -f namespace.yaml
```

#### 2. 部署存储层

FastGPT 需要多个持久化存储，我们先创建 PVC：

**MongoDB PVC**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: fastgpt-mongo
  namespace: fastgpt
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Mi
```

**Redis PVC**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: fastgpt-redis
  namespace: fastgpt
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Mi
```

**MinIO PVC**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: fastgpt-minio
  namespace: fastgpt
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Mi
```

**PostgreSQL PVC**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: fastgpt-pg
  namespace: fastgpt
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Mi
```

**AI Proxy PostgreSQL PVC**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: fastgpt-aiproxy-pg
  namespace: fastgpt
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Mi
```

应用所有 PVC：

```bash
kubectl apply -f pvc/
```

#### 3. 部署数据库服务

##### 3.1 部署 MongoDB

MongoDB 配置为副本集模式，这是 FastGPT 的要求：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fastgpt-mongo
  namespace: fastgpt
spec:
  replicas: 1
  selector:
    matchLabels:
      app: fastgpt-mongo
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: fastgpt-mongo
    spec:
      containers:
        - args:
            - mongod
            - --keyFile
            - /data/mongodb.key
            - --replSet
            - rs0
          command:
            - bash
            - -c
            - |
              openssl rand -base64 128 > /data/mongodb.key
              chmod 400 /data/mongodb.key
              chown 999:999 /data/mongodb.key
              echo 'const isInited = rs.status().ok === 1
              if(!isInited){
                rs.initiate({
                    _id: "rs0",
                    members: [
                        { _id: 0, host: "fastgpt-mongo:27017" }
                    ]
                })
              }' > /data/initReplicaSet.js
              exec docker-entrypoint.sh "$@" &
              until mongo -u myusername -p mypassword --authenticationDatabase admin --eval "print('waited for connection')"; do
                echo "Waiting for MongoDB to start..."
                sleep 2
              done
              mongo -u myusername -p mypassword --authenticationDatabase admin /data/initReplicaSet.js
              wait $!
          env:
            - name: MONGO_INITDB_ROOT_PASSWORD
              value: mypassword
            - name: MONGO_INITDB_ROOT_USERNAME
              value: myusername
          image: registry.cn-hangzhou.aliyuncs.com/fastgpt/mongo:5.0.32
          livenessProbe:
            exec:
              command:
                - mongo
                - -u
                - myusername
                - -p
                - mypassword
                - --authenticationDatabase
                - admin
                - --eval
                - db.adminCommand('ping')
            failureThreshold: 5
            initialDelaySeconds: 30
            periodSeconds: 10
            timeoutSeconds: 5
          name: fastgpt-mongo
          volumeMounts:
            - mountPath: /data/db
              name: fastgpt-mongo
      restartPolicy: Always
      volumes:
        - name: fastgpt-mongo
          persistentVolumeClaim:
            claimName: fastgpt-mongo
```

**MongoDB Service**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: fastgpt-mongo
  namespace: fastgpt
spec:
  ports:
    - name: "27017"
      port: 27017
      targetPort: 27017
  selector:
    app: fastgpt-mongo
```

##### 3.2 部署 Redis

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fastgpt-redis
  namespace: fastgpt
spec:
  replicas: 1
  selector:
    matchLabels:
      app: fastgpt-redis
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: fastgpt-redis
    spec:
      containers:
        - args:
            - redis-server
            - --requirepass
            - mypassword
            - --loglevel
            - warning
            - --maxclients
            - "10000"
            - --appendonly
            - "yes"
            - --save
            - "60"
            - "10"
            - --maxmemory
            - 4gb
            - --maxmemory-policy
            - noeviction
          image: registry.cn-hangzhou.aliyuncs.com/fastgpt/redis:7.2-alpine
          livenessProbe:
            exec:
              command:
                - redis-cli
                - -a
                - mypassword
                - ping
            failureThreshold: 3
            initialDelaySeconds: 30
            periodSeconds: 10
            timeoutSeconds: 3
          name: fastgpt-redis
          volumeMounts:
            - mountPath: /data
              name: fastgpt-redis
      restartPolicy: Always
      volumes:
        - name: fastgpt-redis
          persistentVolumeClaim:
            claimName: fastgpt-redis
```

**Redis Service**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: fastgpt-redis
  namespace: fastgpt
spec:
  ports:
    - name: "6379"
      port: 6379
      targetPort: 6379
  selector:
    app: fastgpt-redis
```

##### 3.3 部署 PostgreSQL（带 pgvector 扩展）

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fastgpt-pg
  namespace: fastgpt
spec:
  replicas: 1
  selector:
    matchLabels:
      app: fastgpt-pg
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: fastgpt-pg
    spec:
      containers:
        - env:
            - name: POSTGRES_DB
              value: postgres
            - name: POSTGRES_PASSWORD
              value: password
            - name: POSTGRES_USER
              value: username
          image: registry.cn-hangzhou.aliyuncs.com/fastgpt/pgvector:0.8.0-pg15
          livenessProbe:
            exec:
              command:
                - pg_isready
                - -U
                - username
                - -d
                - postgres
            failureThreshold: 10
            periodSeconds: 5
            timeoutSeconds: 5
          name: fastgpt-pg
          volumeMounts:
            - mountPath: /var/lib/postgresql/data
              name: fastgpt-pg
      restartPolicy: Always
      volumes:
        - name: fastgpt-pg
          persistentVolumeClaim:
            claimName: fastgpt-pg
```

**PostgreSQL Service**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: fastgpt-pg
  namespace: fastgpt
spec:
  ports:
    - name: "5432"
      port: 5432
      targetPort: 5432
  selector:
    app: fastgpt-pg
```

##### 3.4 部署 MinIO

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fastgpt-minio
  namespace: fastgpt
spec:
  replicas: 1
  selector:
    matchLabels:
      app: fastgpt-minio
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: fastgpt-minio
    spec:
      containers:
        - args:
            - server
            - /data
            - --console-address
            - :9001
          env:
            - name: MINIO_ROOT_PASSWORD
              value: minioadmin
            - name: MINIO_ROOT_USER
              value: minioadmin
          image: registry.cn-hangzhou.aliyuncs.com/fastgpt/minio:RELEASE.2025-09-07T16-13-09Z
          livenessProbe:
            httpGet:
              path: /minio/health/live
              port: 9000
            failureThreshold: 3
            periodSeconds: 30
            timeoutSeconds: 20
          name: fastgpt-minio
          ports:
            - containerPort: 9000
              protocol: TCP
            - containerPort: 9001
              protocol: TCP
          volumeMounts:
            - mountPath: /data
              name: fastgpt-minio
      restartPolicy: Always
      volumes:
        - name: fastgpt-minio
          persistentVolumeClaim:
            claimName: fastgpt-minio
```

**MinIO Service**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: fastgpt-minio
  namespace: fastgpt
spec:
  ports:
    - name: "9000"
      port: 9000
      targetPort: 9000
    - name: "9001"
      port: 9001
      targetPort: 9001
  selector:
    app: fastgpt-minio
```

**MinIO LoadBalancer Service（用于外部访问）**

FastGPT 需要通过 `STORAGE_EXTERNAL_ENDPOINT` 环境变量访问 MinIO 的外部地址。创建一个 LoadBalancer 类型的 Service 来暴露 MinIO：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: fastgpt-minio-lb
  namespace: fastgpt
spec:
  type: LoadBalancer
  ports:
    - name: "9000"
      port: 9000
      targetPort: 9000
  selector:
    app: fastgpt-minio
```

应用 LoadBalancer Service：

```bash
kubectl apply -f fastgpt-minio-service-lb.yaml
```

等待 LoadBalancer 分配外部 IP：

```bash
kubectl get svc fastgpt-minio-lb -n fastgpt
```

输出示例：

```
NAME                TYPE           CLUSTER-IP       EXTERNAL-IP      PORT(S)          AGE
fastgpt-minio-lb    LoadBalancer   10.96.123.45     192.168.2.147    9000/TCP         2m
```

记录下 `EXTERNAL-IP` 的值（例如：`192.168.2.147`），这个地址将用于配置 `STORAGE_EXTERNAL_ENDPOINT` 环境变量。

**注意**：
- 如果使用云服务商的 Kubernetes（如 AWS、GKE、AKS），LoadBalancer 会自动分配公网 IP
- 如果使用本地 Kubernetes（如 minikube、k3s），可能需要使用 NodePort 或 MetalLB 来获取外部 IP
- 如果使用 NodePort，可以使用 `kubectl get nodes -o wide` 获取节点 IP，然后使用 `节点IP:NodePort` 格式

#### 4. 部署应用配置

创建 ConfigMap 来存储 FastGPT 的配置：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fastgpt-app-configmap
  namespace: fastgpt
data:
  config.json: |
    {
      "feConfigs": {
        "lafEnv": "https://laf.dev"
      },
      "systemEnv": {
        "vectorMaxProcess": 15,
        "qaMaxProcess": 15,
        "vlmMaxProcess": 15,
        "tokenWorkers": 50,
        "hnswEfSearch": 100,
        "customPdfParse": {
          "url": "",
          "key": "",
          "doc2xKey": "",
          "price": 0
        }
      }
    }
```

#### 5. 部署 FastGPT 主应用

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fastgpt-app
  namespace: fastgpt
spec:
  replicas: 1
  selector:
    matchLabels:
      app: fastgpt-app
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: fastgpt-app
    spec:
      containers:
        - env:
            - name: AES256_SECRET_KEY
              value: fastgptsecret
            - name: AGENT_SANDBOX_ENABLE_VOLUME
              value: "true"
            - name: AGENT_SANDBOX_OPENSANDBOX_BASEURL
              value: http://opensandbox-server:8090
            - name: AGENT_SANDBOX_OPENSANDBOX_IMAGE_REPO
              value: registry.cn-hangzhou.aliyuncs.com/fastgpt/fastgpt-agent-sandbox
            - name: AGENT_SANDBOX_OPENSANDBOX_IMAGE_TAG
              value: v0.1
            - name: AGENT_SANDBOX_OPENSANDBOX_RUNTIME
              value: docker
            - name: AGENT_SANDBOX_PROVIDER
              value: opensandbox
            - name: AGENT_SANDBOX_VOLUME_MANAGER_TOKEN
              value: vmtoken
            - name: AGENT_SANDBOX_VOLUME_MANAGER_URL
              value: http://fastgpt-volume-manager:3000
            - name: AIPROXY_API_ENDPOINT
              value: http://fastgpt-aiproxy:3000
            - name: AIPROXY_API_TOKEN
              value: token
            - name: CHECK_INTERNAL_IP
              value: "false"
            - name: CODE_SANDBOX_TOKEN
              value: codesandbox
            - name: CODE_SANDBOX_URL
              value: http://fastgpt-code-sandbox:3000
            - name: DB_MAX_LINK
              value: "5"
            - name: DEFAULT_ROOT_PSW
              value: "1234"
            - name: FE_DOMAIN
              value: http://localhost:3000
            - name: FILE_TOKEN_KEY
              value: filetokenkey
            - name: LLM_REQUEST_TRACKING_RETENTION_HOURS
              value: "6"
            - name: LOG_CONSOLE_LEVEL
              value: debug
            - name: LOG_ENABLE_CONSOLE
              value: "true"
            - name: LOG_ENABLE_OTEL
              value: "false"
            - name: LOG_OTEL_LEVEL
              value: info
            - name: LOG_OTEL_SERVICE_NAME
              value: fastgpt-client
            - name: LOG_OTEL_URL
              value: http://localhost:4318/v1/logs
            - name: MAX_HTML_TRANSFORM_CHARS
              value: "1000000"
            - name: MONGODB_URI
              value: mongodb://myusername:mypassword@fastgpt-mongo:27017/fastgpt?authSource=admin
            - name: MULTIPLE_DATA_TO_BASE64
              value: "true"
            - name: PG_URL
              value: postgresql://username:password@fastgpt-pg:5432/postgres
            - name: PLUGIN_BASE_URL
              value: http://fastgpt-plugin:3000
            - name: PLUGIN_TOKEN
              value: token
            - name: REDIS_URL
              value: redis://default:mypassword@fastgpt-redis:6379
            - name: ROOT_KEY
              value: fastgpt-xxx
            - name: SERVICE_REQUEST_MAX_CONTENT_LENGTH
              value: "10"
            - name: STORAGE_ACCESS_KEY_ID
              value: minioadmin
            - name: STORAGE_EXTERNAL_ENDPOINT
              value: http://192.168.2.147:9000
            - name: STORAGE_PRIVATE_BUCKET
              value: fastgpt-private
            - name: STORAGE_PUBLIC_BUCKET
              value: fastgpt-public
            - name: STORAGE_REGION
              value: us-east-1
            - name: STORAGE_S3_ENDPOINT
              value: http://fastgpt-minio:9000
            - name: STORAGE_S3_FORCE_PATH_STYLE
              value: "true"
            - name: STORAGE_S3_MAX_RETRIES
              value: "3"
            - name: STORAGE_SECRET_ACCESS_KEY
              value: minioadmin
            - name: STORAGE_VENDOR
              value: minio
            - name: SYNC_INDEX
              value: "1"
            - name: TOKEN_KEY
              value: fastgpt
            - name: UPLOAD_FILE_MAX_AMOUNT
              value: "1000"
            - name: UPLOAD_FILE_MAX_SIZE
              value: "1000"
            - name: USE_IP_LIMIT
              value: "false"
            - name: WORKFLOW_MAX_LOOP_TIMES
              value: "100"
            - name: WORKFLOW_MAX_RUN_TIMES
              value: "1000"
          image: registry.cn-hangzhou.aliyuncs.com/fastgpt/fastgpt:v4.14.10.4
          name: fastgpt-app
          ports:
            - containerPort: 3000
              protocol: TCP
          volumeMounts:
            - name: fastgpt-app-configmap
              mountPath: /app/data/config.json
              subPath: config.json
      restartPolicy: Always
      volumes:
        - name: fastgpt-app-configmap
          configMap:
            name: fastgpt-app-configmap
```

**FastGPT App Service**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: fastgpt-app
  namespace: fastgpt
spec:
  ports:
    - name: "3000"
      port: 3000
      targetPort: 3000
  selector:
    app: fastgpt-app
```

#### 6. 部署辅助服务

##### 6.1 部署 AI Proxy

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fastgpt-aiproxy
  namespace: fastgpt
spec:
  replicas: 1
  selector:
    matchLabels:
      app: fastgpt-aiproxy
  template:
    metadata:
      labels:
        app: fastgpt-aiproxy
    spec:
      containers:
        - env:
            - name: ADMIN_KEY
              value: token
            - name: BILLING_ENABLED
              value: "false"
            - name: DISABLE_MODEL_CONFIG
              value: "true"
            - name: LOG_DETAIL_STORAGE_HOURS
              value: "1"
            - name: RETRY_TIMES
              value: "3"
            - name: SQL_DSN
              value: postgres://postgres:aiproxy@fastgpt-aiproxy-pg:5432/aiproxy
          image: registry.cn-hangzhou.aliyuncs.com/labring/aiproxy:v0.3.5
          livenessProbe:
            httpGet:
              path: /api/status
              port: 3000
            failureThreshold: 10
            periodSeconds: 5
            timeoutSeconds: 5
          name: fastgpt-aiproxy
      restartPolicy: Always
```

**AI Proxy Service**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: fastgpt-aiproxy
  namespace: fastgpt
spec:
  ports:
    - name: "3000"
      port: 3000
      targetPort: 3000
  selector:
    app: fastgpt-aiproxy
```

##### 6.2 部署 Code Sandbox

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fastgpt-code-sandbox
  namespace: fastgpt
spec:
  replicas: 1
  selector:
    matchLabels:
      app: fastgpt-code-sandbox
  template:
    metadata:
      labels:
        app: fastgpt-code-sandbox
    spec:
      containers:
        - env:
            - name: CHECK_INTERNAL_IP
              value: "false"
            - name: LOG_CONSOLE_LEVEL
              value: debug
            - name: LOG_ENABLE_CONSOLE
              value: "true"
            - name: LOG_ENABLE_OTEL
              value: "false"
            - name: LOG_OTEL_LEVEL
              value: info
            - name: LOG_OTEL_SERVICE_NAME
              value: fastgpt-code-sandbox
            - name: LOG_OTEL_URL
              value: http://localhost:4318/v1/logs
            - name: SANDBOX_JS_ALLOWED_MODULES
              value: lodash,dayjs,moment,uuid,crypto-js,qs,url,querystring
            - name: SANDBOX_MAX_MEMORY_MB
              value: "256"
            - name: SANDBOX_MAX_TIMEOUT
              value: "60000"
            - name: SANDBOX_POOL_SIZE
              value: "20"
            - name: SANDBOX_PYTHON_ALLOWED_MODULES
              value: math,cmath,decimal,fractions,random,statistics,collections,array,heapq,bisect,queue,copy,itertools,functools,operator,string,re,difflib,textwrap,unicodedata,codecs,datetime,time,calendar,_strptime,json,csv,base64,binascii,struct,hashlib,hmac,secrets,uuid,typing,abc,enum,dataclasses,contextlib,pprint,weakref,numpy,pandas,matplotlib
            - name: SANDBOX_REQUEST_MAX_BODY_MB
              value: "5"
            - name: SANDBOX_REQUEST_MAX_COUNT
              value: "30"
            - name: SANDBOX_REQUEST_MAX_RESPONSE_MB
              value: "10"
            - name: SANDBOX_REQUEST_TIMEOUT
              value: "60000"
            - name: SANDBOX_TOKEN
              value: codesandbox
          image: registry.cn-hangzhou.aliyuncs.com/fastgpt/fastgpt-code-sandbox:v4.14.10
          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            failureThreshold: 3
            periodSeconds: 30
            timeoutSeconds: 20
          name: fastgpt-code-sandbox
      restartPolicy: Always
```

##### 6.3 部署 MCP Server

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fastgpt-mcp-server
  namespace: fastgpt
spec:
  replicas: 1
  selector:
    matchLabels:
      app: fastgpt-mcp-server
  template:
    metadata:
      labels:
        app: fastgpt-mcp-server
    spec:
      containers:
        - env:
            - name: FASTGPT_ENDPOINT
              value: http://fastgpt-app:3000
            - name: LOG_CONSOLE_LEVEL
              value: debug
            - name: LOG_ENABLE_CONSOLE
              value: "true"
            - name: LOG_ENABLE_OTEL
              value: "false"
            - name: LOG_OTEL_LEVEL
              value: info
            - name: LOG_OTEL_URL
              value: http://localhost:4318/v1/logs
          image: registry.cn-hangzhou.aliyuncs.com/fastgpt/fastgpt-mcp_server:v4.14.10
          name: fastgpt-mcp-server
          ports:
            - containerPort: 3000
              protocol: TCP
      restartPolicy: Always
```

**MCP Server Service**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: fastgpt-mcp-server
  namespace: fastgpt
spec:
  ports:
    - name: "3003"
      port: 3003
      targetPort: 3000
  selector:
    app: fastgpt-mcp-server
```

##### 6.4 部署 Plugin Service

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fastgpt-plugin
  namespace: fastgpt
spec:
  replicas: 1
  selector:
    matchLabels:
      app: fastgpt-plugin
  template:
    metadata:
      labels:
        app: fastgpt-plugin
    spec:
      containers:
        - env:
            - name: AUTH_TOKEN
              value: token
            - name: DB_MAX_LINK
              value: "100"
            - name: LOG_CONSOLE_LEVEL
              value: debug
            - name: LOG_ENABLE_CONSOLE
              value: "true"
            - name: LOG_ENABLE_OTEL
              value: "false"
            - name: LOG_OTEL_LEVEL
              value: info
            - name: LOG_OTEL_SERVICE_NAME
              value: fastgpt-plugin
            - name: LOG_OTEL_URL
              value: http://localhost:4318/v1/logs
            - name: MAX_API_SIZE
              value: "10"
            - name: MONGODB_URI
              value: mongodb://myusername:mypassword@fastgpt-mongo:27017/fastgpt?authSource=admin
            - name: REDIS_URL
              value: redis://default:mypassword@fastgpt-redis:6379
            - name: SERVICE_REQUEST_MAX_CONTENT_LENGTH
              value: "10"
            - name: STORAGE_ACCESS_KEY_ID
              value: minioadmin
            - name: STORAGE_EXTERNAL_ENDPOINT
              value: http://192.168.0.2:9000
            - name: STORAGE_PRIVATE_BUCKET
              value: fastgpt-private
            - name: STORAGE_PUBLIC_BUCKET
              value: fastgpt-public
            - name: STORAGE_REGION
              value: us-east-1
            - name: STORAGE_S3_ENDPOINT
              value: http://fastgpt-minio:9000
            - name: STORAGE_S3_FORCE_PATH_STYLE
              value: "true"
            - name: STORAGE_S3_MAX_RETRIES
              value: "3"
            - name: STORAGE_SECRET_ACCESS_KEY
              value: minioadmin
            - name: STORAGE_VENDOR
              value: minio
          image: registry.cn-hangzhou.aliyuncs.com/fastgpt/fastgpt-plugin:v0.5.6
          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            failureThreshold: 3
            periodSeconds: 30
            timeoutSeconds: 20
          name: fastgpt-plugin
      restartPolicy: Always
```

**Plugin Service**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: fastgpt-plugin
  namespace: fastgpt
spec:
  ports:
    - name: "3000"
      port: 3000
      targetPort: 3000
  selector:
    app: fastgpt-plugin
```

##### 6.5 部署 Volume Manager

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fastgpt-volume-manager
  namespace: fastgpt
spec:
  replicas: 1
  selector:
    matchLabels:
      app: fastgpt-volume-manager
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: fastgpt-volume-manager
    spec:
      containers:
        - env:
            - name: PORT
              value: "3000"
            - name: VM_AUTH_TOKEN
              value: vmtoken
            - name: VM_DOCKER_API_VERSION
              value: v1.44
            - name: VM_LOG_LEVEL
              value: info
            - name: VM_RUNTIME
              value: docker
            - name: VM_VOLUME_NAME_PREFIX
              value: fastgpt-session
          image: registry.cn-hangzhou.aliyuncs.com/fastgpt/fastgpt-agent-volume-manager:v0.1
          livenessProbe:
            exec:
              command:
                - bun
                - -e
                - fetch('http://localhost:3000/health').then((res) => { if (!res.ok) throw new Error(String(res.status)); })
            failureThreshold: 5
            periodSeconds: 10
            timeoutSeconds: 5
          name: fastgpt-volume-manager
      restartPolicy: Always
```

#### 7. 部署 AI Proxy PostgreSQL

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fastgpt-aiproxy-pg
  namespace: fastgpt
spec:
  replicas: 1
  selector:
    matchLabels:
      app: fastgpt-aiproxy-pg
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: fastgpt-aiproxy-pg
    spec:
      containers:
        - env:
            - name: POSTGRES_DB
              value: aiproxy
            - name: POSTGRES_PASSWORD
              value: aiproxy
            - name: POSTGRES_USER
              value: postgres
            - name: TZ
              value: Asia/Shanghai
          image: registry.cn-hangzhou.aliyuncs.com/fastgpt/pgvector:0.8.0-pg15
          livenessProbe:
            exec:
              command:
                - pg_isready
                - -U
                - postgres
                - -d
                - aiproxy
            failureThreshold: 10
            periodSeconds: 5
            timeoutSeconds: 5
          name: fastgpt-aiproxy-pg
          volumeMounts:
            - mountPath: /var/lib/postgresql/data
              name: fastgpt-aiproxy-pg
      restartPolicy: Always
      volumes:
        - name: fastgpt-aiproxy-pg
          persistentVolumeClaim:
            claimName: fastgpt-aiproxy-pg
```

**AI Proxy PostgreSQL Service**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: fastgpt-aiproxy-pg
  namespace: fastgpt
spec:
  ports:
    - name: "5432"
      port: 5432
      targetPort: 5432
  selector:
    app: fastgpt-aiproxy-pg
```

#### 8. 验证部署

等待所有 Pod 启动完成：

```bash
kubectl get pods -n fastgpt
```

检查所有服务状态：

```bash
kubectl get svc -n fastgpt
```

查看日志：

```bash
kubectl logs -f deployment/fastgpt-app -n fastgpt
```

#### 9. 访问 FastGPT

创建 Ingress 或使用 NodePort/LoadBalancer 来暴露 FastGPT 服务：

**使用 NodePort**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: fastgpt-app-nodeport
  namespace: fastgpt
spec:
  type: NodePort
  ports:
    - port: 3000
      targetPort: 3000
      nodePort: 30080
  selector:
    app: fastgpt-app
```

访问地址：`http://<节点IP>:30080`

默认登录账号：
- 用户名：root
- 密码：1234

### 配置说明

#### 环境变量配置

FastGPT 主应用的主要环境变量：

| 变量名 | 说明 | 默认值 |
|--------|------|--------|
| `MONGODB_URI` | MongoDB 连接字符串 | - |
| `PG_URL` | PostgreSQL 连接字符串 | - |
| `REDIS_URL` | Redis 连接字符串 | - |
| `STORAGE_S3_ENDPOINT` | MinIO/S3 端点 | - |
| `STORAGE_ACCESS_KEY_ID` | 存储访问密钥 | - |
| `STORAGE_SECRET_ACCESS_KEY` | 存储密钥 | - |
| `ROOT_KEY` | 根密钥 | fastgpt-xxx |
| `DEFAULT_ROOT_PSW` | 默认管理员密码 | 1234 |
| `FE_DOMAIN` | 前端域名 | http://localhost:3000 |

#### 存储配置

建议根据实际需求调整 PVC 的大小：

- **MongoDB**: 建议至少 10GB
- **Redis**: 建议至少 5GB
- **MinIO**: 建议至少 50GB（根据文件存储需求）
- **PostgreSQL**: 建议至少 10GB

#### 资源限制

建议为每个组件添加资源限制：

```yaml
resources:
  requests:
    memory: "256Mi"
    cpu: "250m"
  limits:
    memory: "512Mi"
    cpu: "500m"
```

### 运维管理

#### 查看日志

```bash
# 查看 FastGPT 应用日志
kubectl logs -f deployment/fastgpt-app -n fastgpt

# 查看 MongoDB 日志
kubectl logs -f deployment/fastgpt-mongo -n fastgpt

# 查看所有 Pod 日志
kubectl logs -f -n fastgpt --all-containers=true
```

#### 扩容

```bash
# 扩容 FastGPT 应用
kubectl scale deployment fastgpt-app -n fastgpt --replicas=3

# 扩容 Code Sandbox
kubectl scale deployment fastgpt-code-sandbox -n fastgpt --replicas=3
```

#### 更新镜像

```bash
# 更新 FastGPT 镜像
kubectl set image deployment/fastgpt-app fastgpt-app=registry.cn-hangzhou.aliyuncs.com/fastgpt/fastgpt:v4.14.10.5 -n fastgpt

# 查看更新状态
kubectl rollout status deployment/fastgpt-app -n fastgpt
```

#### 备份

```bash
# 备份 MongoDB
kubectl exec -n fastgpt deployment/fastgpt-mongo -- mongodump --uri="mongodb://myusername:mypassword@localhost:27017/fastgpt?authSource=admin" --archive=/data/mongodb-backup.gz
kubectl cp -n fastgpt fastgpt-mongo-xxx:/data/mongodb-backup.gz ./mongodb-backup.gz

# 备份 PostgreSQL
kubectl exec -n fastgpt deployment/fastgpt-pg -- pg_dump -U username postgres > postgres-backup.sql
```

#### 恢复

```bash
# 恢复 MongoDB
kubectl cp ./mongodb-backup.gz -n fastgpt fastgpt-mongo-xxx:/data/mongodb-backup.gz
kubectl exec -n fastgpt deployment/fastgpt-mongo -- mongorestore --uri="mongodb://myusername:mypassword@localhost:27017/fastgpt?authSource=admin" --archive=/data/mongodb-backup.gz

# 恢复 PostgreSQL
cat postgres-backup.sql | kubectl exec -i -n fastgpt deployment/fastgpt-pg -- psql -U username -d postgres
```

### 故障排查

#### Pod 无法启动

```bash
# 查看 Pod 状态
kubectl describe pod <pod-name> -n fastgpt

# 查看 Pod 日志
kubectl logs <pod-name> -n fastgpt

# 查看事件
kubectl get events -n fastgpt --sort-by='.lastTimestamp'
```

#### 服务无法访问

```bash
# 检查 Service
kubectl get svc -n fastgpt

# 检查 Endpoints
kubectl get endpoints -n fastgpt

# 测试服务连通性
kubectl run -it --rm debug --image=nicolaka/netshoot --restart=Never -n fastgpt -- curl http://fastgpt-app:3000
```

#### 数据库连接问题

```bash
# 测试 MongoDB 连接
kubectl run -it --rm mongo-test --image=mongo:5.0 --restart=Never -n fastgpt -- mongo --host fastgpt-mongo --port 27017 -u myusername -p mypassword --authenticationDatabase admin

# 测试 Redis 连接
kubectl run -it --rm redis-test --image=redis:7.2 --restart=Never -n fastgpt -- redis-cli -h fastgpt-redis -p 6379 -a mypassword ping

# 测试 PostgreSQL 连接
kubectl run -it --rm pg-test --image=postgres:15 --restart=Never -n fastgpt -- psql -h fastgpt-pg -p 5432 -U username -d postgres
```

### 清理

如果需要删除 FastGPT 部署：

```bash
# 删除所有资源
kubectl delete namespace fastgpt

# 或者逐个删除
kubectl delete -f . -n fastgpt
```

### 总结

本文详细介绍了在 Kubernetes 环境中部署 FastGPT 的完整过程，包括：

1. 架构概述和组件说明
2. 详细的部署步骤和配置
3. 运维管理和故障排查方法
4. 备份恢复和扩容策略

通过 Kubernetes 部署 FastGPT，可以获得以下优势：

- **高可用性**: 通过副本和自动重启保证服务可用性
- **易扩展性**: 可以根据负载动态调整资源
- **易维护性**: 统一的配置管理和日志收集
- **资源隔离**: 通过命名空间实现资源隔离
- **滚动更新**: 支持零停机更新

希望本文能帮助你在 Kubernetes 环境中成功部署 FastGPT，构建强大的 AI 知识库系统。
