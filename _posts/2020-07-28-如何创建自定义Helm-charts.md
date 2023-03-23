---
layout:     post
title:      如何创建自定义Helm charts
subtitle:   为OpenProject创建一个具有数据持久性和外部数据库的自定义Helm charts
date:       2020-07-28
author:     J
catalog: true
tags:
    - kubernetes	
    - Helm
---



#### TL；DR 摘要标题

> 前提确保helm已安装 关于如何安装helm可[参考这里](https://github.com/helm/helm)

- `helm create [chart name]`
- 编辑`value.yaml`更改要部署的Docker镜像
- 使用`templates`文件夹中的任何K8s功能进行自定义
- 使用`helm install [release name] [chart folder]`部署

> 默认使用helm 3来进行包管理

### Helm Create

首先，我们将使用`helm create`命令来创建我们的自定义图表。我将使用[OpenProject](https://www.openproject.org/)作为本文的首选应用程序。您可以使用任何喜欢的应用程序，但要注意的是，它需要在注册表上托管Docker映像。

> 请注意，每个应用程序都有自己的细微差别和要求。如果选择其他应用程序，请确保阅读其有关部署要求的文档。

我们将为OpenProject创建一个项目管理软件的helm chart。首先，使用`helm create`命令创建项目的基线。

```shell
helm create openproject-helm-chart
```

我们将看到以下helm chart的目录结构

```shell
root@node1:/tmp/openproject-helm-chart# tree
.
├── charts
├── Chart.yaml
├── templates
│?? ├── deployment.yaml
│?? ├── _helpers.tpl
│?? ├── ingress.yaml
│?? ├── NOTES.txt
│?? ├── serviceaccount.yaml
│?? ├── service.yaml
│?? └── tests
│??     └── test-connection.yaml
└── values.yaml
```

所有kubernetes规范文件都包含在该`templates`文件夹中。它们都使用[Go Template Engine](https://golang.org/pkg/text/template/)进行了[模板化](https://golang.org/pkg/text/template/)。在这篇文章中，我们将主要集中在`chart.yaml`，`deployment.yaml`和`values.yaml`文件。

首先，我们将需要修改`values.yaml`文件以更新用于该图表的Docker映像。在这里，我们还将更改为`service.type`，`LoadBalancer`因此我们可以从外部访问我们的OpenProject实例。

> 建议保留`service.type`as `ClusterIP`并用于`ingress`将流量定向到内部实例。

你的value.yaml 文件应如下所示：

```yaml
# Default values for openproject-helm-chart.
# This is a YAML-formatted file.
# Declare variables to be passed into your templates.

replicaCount: 1

image:
  repository: openproject/community
  pullPolicy: IfNotPresent
  # Overrides the image tag whose default is the chart appVersion.
  tag: "1.16.0"

imagePullSecrets: []
nameOverride: ""
fullnameOverride: ""

serviceAccount:
  # Specifies whether a service account should be created
  create: true
  # Annotations to add to the service account
  annotations: {}
  # The name of the service account to use.
  # If not set and create is true, a name is generated using the fullname template
  name:

podSecurityContext: {}
  # fsGroup: 2000

securityContext: {}
  # capabilities:
  #   drop:
  #   - ALL
  # readOnlyRootFilesystem: true
  # runAsNonRoot: true
  # runAsUser: 1000

service:
  type: ClusterIP
  port: 8080

ingress:
  enabled: false
  annotations: {}
    # kubernetes.io/ingress.class: nginx
    # kubernetes.io/tls-acme: "true"
  hosts:
    - host: chart-example.local
      paths: []
  tls: []
  #  - secretName: chart-example-tls
  #    hosts:
  #      - chart-example.local

resources: {}
  # We usually recommend not to specify default resources and to leave this as a conscious
  # choice for the user. This also increases chances charts run on environments with little
  # resources, such as Minikube. If you do want to specify resources, uncomment the following
  # lines, adjust them as necessary, and remove the curly braces after 'resources:'.
  # limits:
  #   cpu: 100m
  #   memory: 128Mi
  # requests:
  #   cpu: 100m
  #   memory: 128Mi

nodeSelector: {}

tolerations: []

affinity: {}
```

你的Chart.yaml 文件如下所示：

```yaml
apiVersion: v2
name: openproject-helm-chart
description: A Helm chart for Kubernetes

# A chart can be either an 'application' or a 'library' chart.
#
# Application charts are a collection of templates that can be packaged into versioned archives
# to be deployed.
#
# Library charts provide useful utilities or functions for the chart developer. They're included as
# a dependency of application charts to inject those utilities and functions into the rendering
# pipeline. Library charts do not define any templates and therefore cannot be deployed.
type: application

# This is the chart version. This version number should be incremented each time you make changes
# to the chart and its templates, including the app version.
version: 0.1.0

# This is the version number of the application being deployed. This version number should be
# incremented each time you make changes to the application.
appVersion: 1.16.0
```

### 数据持久化存储

如果默认不做数据持久化，是用`emptyDir`类型，这个是跟pod的生命周期一致，无法在pod删除或者重建的时候数据保留下来，所以这里就有数据持久化存储的必要性。

因此，首先我们将pvc.yaml文件添加到template的文件夹中

```yaml
{{- if and .Values.persistence.enabled (not .Values.persistence.existingClaim) }}
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: {{ template "openproject-helm-chart.fullname" . }}
  labels: {{- include "openproject-helm-chart.labels" . | nindent 4 }}
spec:
  accessModes:
    - {{ .Values.persistence.accessMode | quote }}
  resources:
    requests:
      storage: {{ .Values.persistence.size | quote }}
{{- end }}
```

现在我们将更新`deployment.yaml`和`value.yaml`文件，以使用此模版。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "openproject-helm-chart.fullname" . }}
  labels:
    {{- include "openproject-helm-chart.labels" . | nindent 4 }}
spec:
{{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
{{- end }}
  selector:
    matchLabels:
      {{- include "openproject-helm-chart.selectorLabels" . | nindent 6 }}
  template:
    metadata:
    {{- with .Values.podAnnotations }}
      annotations:
        {{- toYaml . | nindent 8 }}
    {{- end }}
      labels:
        {{- include "openproject-helm-chart.selectorLabels" . | nindent 8 }}
    spec:
      {{- with .Values.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      serviceAccountName: {{ include "openproject-helm-chart.serviceAccountName" . }}
      securityContext:
        {{- toYaml .Values.podSecurityContext | nindent 8 }}
      containers:
        - name: {{ .Chart.Name }}
          securityContext:
            {{- toYaml .Values.securityContext | nindent 12 }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: 8080
              protocol: TCP
          livenessProbe:
            httpGet:
              path: /
              port: http
          readinessProbe:
            httpGet:
              path: /
              port: http
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
          volumeMounts:
            - mountPath: /var/openproject
              name: openproject-data
      volumes:
        - name: openproject-data
        {{- if .Values.persistence.enabled }}
          persistentVolumeClaim:
            claimName: {{ .Values.persistence.existingClaim | default (include "openproject-helm-chart.fullname" .) }}
        {{- else }}
          emptyDir: {}
        {{ end }}
      {{- with .Values.nodeSelector }}
      nodeSelector:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.affinity }}
      affinity:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.tolerations }}
      tolerations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
```

**values.yaml**：添加了**数据持久化部分**

```yaml
...
persistence:
  enabled: true
  accessMode: ReadWriteOnce
  size: 48Gi
...
```

这将为我们的OpenProject实例添加48 GB的卷。它将安装在`/var/openproject`的路径下。

### 添加外部数据库

要将外部数据库添加到我们的OpenProject实例中，我们需要向我们的图表添加一个依赖项。加上`_helpers.tpl`使用帮助程序功能更新`deployment.yaml`和，并更新和`values.yaml`文件以使用外部数据库。首先，我们将更新`chart.yaml`文件以将PostgreSQL添加为依赖项。我们将为PostgreSQL使用出色的Bitnami图表。您的`chart.yaml`文件应如下所示：

```yaml
apiVersion: v2
name: openproject-helm-chart
description: A Helm chart for Kubernetes

# A chart can be either an 'application' or a 'library' chart.
#
# Application charts are a collection of templates that can be packaged into versioned archives
# to be deployed.
#
# Library charts provide useful utilities or functions for the chart developer. They're included as
# a dependency of application charts to inject those utilities and functions into the rendering
# pipeline. Library charts do not define any templates and therefore cannot be deployed.
type: application

# This is the chart version. This version number should be incremented each time you make changes
# to the chart and its templates, including the app version.
version: 0.1.0

# This is the version number of the application being deployed. This version number should be
# incremented each time you make changes to the application.
appVersion: 1.16.0

dependencies:
- name: postgresql
  version: "9.1.1"
  repository: "https://charts.bitnami.com/bitnami"
```

现在我们必须更新`_helpers.tpl`文件以将以下部分添加到文件中

```yaml
{{/*
Create a default fully qualified app name.
We truncate at 63 chars because some Kubernetes name fields are limited to this (by the DNS naming spec).
*/}}
{{- define "openproject-helm-chart.postgresql.fullname" -}}
{{- printf "%s-%s" .Release.Name "postgresql" | trunc 63 | trimSuffix "-" -}}
{{- end -}}
```

**values.yaml**：添加了**Postgresql部分**

```yaml
...
postgresql:
  persistence:
    enabled: true
    size: 15G
  postgresqlPassword: superSecretPassword
  postgresqlDatabase: openproject
...
```

通过上述更改，我们现在在应用程序外部建立了一个数据库。

### 部署

经过以上更改，我们有了OpenProject的工作表。假设您已经设置了kubernetes集群，那么我们可以通过几个简单的步骤来部署：

1. `helm dependency update`
2. `helm install openproject ./openproject-helm-chart`