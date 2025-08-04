# Helm 高级功能指南

本文档深入介绍 Helm 的核心功能：Override Values、--dry-run、--debug 和 get 命令。从基础概念到实际应用，帮助您熟练掌握这些重要工具。

## 目录
- [Helm Chart 目录结构](#helm-chart-目录结构)
- [Helm 内置对象 (Builtin Objects)](#helm-内置对象-builtin-objects)
- [Helm 开发基础 (Development Basics)](#helm-开发基础-development-basics)
- [Helm 高级控制结构 (Advanced Control Structures)](#helm-高级控制结构-advanced-control-structures)
- [Override Values (值覆盖)](#override-values-值覆盖)
- [--dry-run (模拟运行)](#--dry-run-模拟运行)
- [--debug (调试模式)](#--debug-调试模式)
- [helm get (获取资源信息)](#helm-get-获取资源信息)
- [综合实践案例](#综合实践案例)
- [最佳实践](#最佳实践)

---

## Helm Chart 目录结构

### 什么是 Helm Chart？

Helm Chart 是用于定义、安装和升级 Kubernetes 应用程序的包格式。Chart 是一个目录，包含了描述相关 Kubernetes 资源集合的文件。

### 为什么需要了解 Chart 结构？

1. **标准化**：统一的目录结构便于理解和维护
2. **模板化**：通过模板实现配置的灵活性
3. **可复用性**：标准结构使 Chart 易于分享和复用
4. **版本管理**：清晰的结构支持版本控制和发布

### Chart 目录结构详解

#### 1. 基础 Chart 结构

```
mychart/                    # Chart 根目录
├── Chart.yaml             # Chart 元数据文件（必需）
├── values.yaml             # 默认配置值（推荐）
├── templates/              # 模板目录（必需）
│   ├── deployment.yaml     # Deployment 模板
│   ├── service.yaml        # Service 模板
│   ├── ingress.yaml        # Ingress 模板
│   ├── configmap.yaml      # ConfigMap 模板
│   ├── secret.yaml         # Secret 模板
│   ├── _helpers.tpl        # 模板助手函数
│   ├── NOTES.txt           # 安装后的使用说明
│   └── tests/              # 测试文件目录
│       └── test-connection.yaml
├── charts/                 # 依赖 Chart 目录
├── crds/                   # 自定义资源定义
├── .helmignore             # Helm 忽略文件
└── README.md               # Chart 说明文档
```

#### 2. 文件和目录详解

##### Chart.yaml（必需）

Chart.yaml 是 Chart 的核心元数据文件：

```yaml
# Chart.yaml 示例
apiVersion: v2                    # Chart API 版本（v2 for Helm 3）
name: webapp                      # Chart 名称
description: A Helm chart for web application  # 简短描述
type: application                 # Chart 类型（application 或 library）
version: 0.1.0                   # Chart 版本（SemVer）
appVersion: "1.16.0"             # 应用版本

# 可选字段
keywords:                         # 关键词列表
  - web
  - nginx
  - frontend
home: https://example.com         # 项目主页
sources:                          # 源码仓库
  - https://github.com/example/webapp
maintainers:                      # 维护者信息
  - name: John Doe
    email: john@example.com
    url: https://johndoe.com
icon: https://example.com/icon.png  # Chart 图标
deprecated: false                 # 是否已弃用
annotations:                      # 注解
  category: Infrastructure
```

##### values.yaml（推荐）

默认配置值文件，定义了 Chart 的所有可配置参数：

```yaml
# values.yaml 示例
# 镜像配置
image:
  repository: nginx
  pullPolicy: IfNotPresent
  tag: "1.20"

# 应用配置
replicaCount: 1
nameOverride: ""
fullnameOverride: ""

# 服务配置
service:
  type: ClusterIP
  port: 80
  targetPort: 8080

# Ingress 配置
ingress:
  enabled: false
  className: ""
  annotations: {}
    # kubernetes.io/ingress.class: nginx
    # cert-manager.io/cluster-issuer: letsencrypt-prod
  hosts:
    - host: chart-example.local
      paths:
        - path: /
          pathType: Prefix
  tls: []

# 资源限制
resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 250m
    memory: 256Mi

# 自动扩缩容
autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 100
  targetCPUUtilizationPercentage: 80

# 节点选择
nodeSelector: {}
tolerations: []
affinity: {}

# 安全上下文
podSecurityContext: {}
securityContext: {}

# 服务账户
serviceAccount:
  create: true
  annotations: {}
  name: ""

# 探针配置
livenessProbe:
  httpGet:
    path: /health
    port: http
  initialDelaySeconds: 30
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /ready
    port: http
  initialDelaySeconds: 5
  periodSeconds: 5
```

##### templates/ 目录

包含所有 Kubernetes 资源模板文件：

**deployment.yaml 示例：**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "webapp.fullname" . }}
  labels:
    {{- include "webapp.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "webapp.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
      labels:
        {{- include "webapp.selectorLabels" . | nindent 8 }}
    spec:
      serviceAccountName: {{ include "webapp.serviceAccountName" . }}
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
              containerPort: {{ .Values.service.targetPort }}
              protocol: TCP
          livenessProbe:
            {{- toYaml .Values.livenessProbe | nindent 12 }}
          readinessProbe:
            {{- toYaml .Values.readinessProbe | nindent 12 }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

**service.yaml 示例：**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "webapp.fullname" . }}
  labels:
    {{- include "webapp.labels" . | nindent 4 }}
spec:
  type: {{ .Values.service.type }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: http
      protocol: TCP
      name: http
  selector:
    {{- include "webapp.selectorLabels" . | nindent 4 }}
```

**_helpers.tpl 示例：**
```yaml
{{/*
Expand the name of the chart.
*/}}
{{- define "webapp.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Create a default fully qualified app name.
*/}}
{{- define "webapp.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- if contains $name .Release.Name }}
{{- .Release.Name | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
{{- end }}

{{/*
Create chart name and version as used by the chart label.
*/}}
{{- define "webapp.chart" -}}
{{- printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Common labels
*/}}
{{- define "webapp.labels" -}}
helm.sh/chart: {{ include "webapp.chart" . }}
{{ include "webapp.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/*
Selector labels
*/}}
{{- define "webapp.selectorLabels" -}}
app.kubernetes.io/name: {{ include "webapp.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}

{{/*
Create the name of the service account to use
*/}}
{{- define "webapp.serviceAccountName" -}}
{{- if .Values.serviceAccount.create }}
{{- default (include "webapp.fullname" .) .Values.serviceAccount.name }}
{{- else }}
{{- default "default" .Values.serviceAccount.name }}
{{- end }}
{{- end }}
```

**NOTES.txt 示例：**
```
1. Get the application URL by running these commands:
{{- if .Values.ingress.enabled }}
{{- range $host := .Values.ingress.hosts }}
  {{- range .paths }}
  http{{ if $.Values.ingress.tls }}s{{ end }}://{{ $host.host }}{{ .path }}
  {{- end }}
{{- end }}
{{- else if contains "NodePort" .Values.service.type }}
  export NODE_PORT=$(kubectl get --namespace {{ .Release.Namespace }} -o jsonpath="{.spec.ports[0].nodePort}" services {{ include "webapp.fullname" . }})
  export NODE_IP=$(kubectl get nodes --namespace {{ .Release.Namespace }} -o jsonpath="{.items[0].status.addresses[0].address}")
  echo http://$NODE_IP:$NODE_PORT
{{- else if contains "LoadBalancer" .Values.service.type }}
     NOTE: It may take a few minutes for the LoadBalancer IP to be available.
           You can watch the status of by running 'kubectl get --namespace {{ .Release.Namespace }} svc -w {{ include "webapp.fullname" . }}'
  export SERVICE_IP=$(kubectl get svc --namespace {{ .Release.Namespace }} {{ include "webapp.fullname" . }} --template "{{"{{ range (index .status.loadBalancer.ingress 0) }}{{.}}{{ end }}"}}")
  echo http://$SERVICE_IP:{{ .Values.service.port }}
{{- else if contains "ClusterIP" .Values.service.type }}
  export POD_NAME=$(kubectl get pods --namespace {{ .Release.Namespace }} -l "app.kubernetes.io/name={{ include "webapp.name" . }},app.kubernetes.io/instance={{ .Release.Name }}" -o jsonpath="{.items[0].metadata.name}")
  export CONTAINER_PORT=$(kubectl get pod --namespace {{ .Release.Namespace }} $POD_NAME -o jsonpath="{.spec.containers[0].ports[0].containerPort}")
  echo "Visit http://127.0.0.1:8080 to use your application"
  kubectl --namespace {{ .Release.Namespace }} port-forward $POD_NAME 8080:$CONTAINER_PORT
{{- end }}
```

##### 其他重要文件

**.helmignore 示例：**
```
# Patterns to ignore when building packages.
# This supports shell glob matching, relative path matching, and
# negation (prefixed with !). Only one pattern per line.
.DS_Store
# Common VCS dirs
.git/
.gitignore
.bzr/
.bzrignore
.hg/
.hgignore
.svn/
# Common backup files
*.swp
*.bak
*.tmp
*.orig
*~
# Various IDEs
.project
.idea/
*.tmproj
.vscode/
```

#### 3. 高级 Chart 结构

```
advanced-chart/
├── Chart.yaml
├── Chart.lock                    # 依赖锁文件
├── values.yaml
├── values.schema.json             # Values 验证 Schema
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── configmap.yaml
│   ├── secret.yaml
│   ├── serviceaccount.yaml
│   ├── rbac.yaml                  # RBAC 配置
│   ├── pdb.yaml                   # Pod Disruption Budget
│   ├── hpa.yaml                   # Horizontal Pod Autoscaler
│   ├── networkpolicy.yaml         # 网络策略
│   ├── _helpers.tpl
│   ├── NOTES.txt
│   └── tests/
│       ├── test-connection.yaml
│       └── test-deployment.yaml
├── charts/                        # 依赖子 Chart
│   └── postgresql-12.1.9.tgz
├── crds/                          # 自定义资源定义
│   └── crd-webapp.yaml
├── files/                         # 静态文件
│   ├── config/
│   │   └── app.conf
│   └── scripts/
│       └── init.sh
├── .helmignore
├── README.md
├── LICENSE
└── CHANGELOG.md
```

#### 4. 模板语法基础

##### 内置对象
```yaml
# .Values - values.yaml 中的值
{{ .Values.image.repository }}

# .Chart - Chart.yaml 中的信息
{{ .Chart.Name }}
{{ .Chart.Version }}

# .Release - 发布相关信息
{{ .Release.Name }}
{{ .Release.Namespace }}

# .Template - 当前模板信息
{{ .Template.Name }}
{{ .Template.BasePath }}

# .Files - 访问 Chart 中的文件
{{ .Files.Get "config/app.conf" }}
```

##### 控制结构
```yaml
# 条件判断
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
# ...
{{- end }}

# 循环
{{- range .Values.ingress.hosts }}
- host: {{ .host }}
  http:
    paths:
    {{- range .paths }}
    - path: {{ .path }}
      pathType: {{ .pathType }}
    {{- end }}
{{- end }}

# with 语句
{{- with .Values.nodeSelector }}
nodeSelector:
  {{- toYaml . | nindent 8 }}
{{- end }}
```

##### 函数和管道
```yaml
# 字符串函数
{{ .Values.image.repository | upper }}
{{ .Values.service.port | quote }}

# 转换函数
{{- toYaml .Values.resources | nindent 12 }}
{{- toJson .Values.config | nindent 4 }}

# 默认值
{{ .Values.image.tag | default .Chart.AppVersion }}

# 包含模板
{{- include "webapp.labels" . | nindent 4 }}
```

### Chart 创建和管理

#### 1. 创建新 Chart

```bash
# 创建标准 Chart 结构
helm create webapp

# 从现有 Chart 创建
helm create webapp --starter mychart

# 查看生成的结构
tree webapp/
```

#### 2. 验证 Chart

```bash
# 语法检查
helm lint ./webapp

# 模板渲染检查
helm template webapp ./webapp

# 详细验证
helm template webapp ./webapp --debug
```

#### 3. 测试 Chart

```bash
# 干运行测试
helm install webapp ./webapp --dry-run --debug

# 运行内置测试
helm test webapp
```

#### 4. 打包和分发

```bash
# 打包 Chart
helm package ./webapp

# 创建索引
helm repo index .

# 推送到仓库
helm push webapp-0.1.0.tgz myrepo
```

### Chart 最佳实践

#### 1. 目录组织

```bash
# ✅ 推荐的文件组织
charts/webapp/
├── Chart.yaml              # 必需，包含完整元数据
├── values.yaml             # 必需，包含所有默认值
├── values.schema.json      # 推荐，验证 values
├── templates/
│   ├── _helpers.tpl        # 必需，通用模板函数
│   ├── deployment.yaml     # 核心资源
│   ├── service.yaml        
│   ├── configmap.yaml      
│   ├── secret.yaml         
│   ├── ingress.yaml        
│   ├── NOTES.txt           # 必需，使用说明
│   └── tests/              # 推荐，测试文件
├── .helmignore             # 推荐
└── README.md               # 必需，文档说明
```

#### 2. 命名约定

```yaml
# ✅ 好的命名实践
# 使用 kebab-case
name: web-app
# 模板文件使用描述性名称
templates/web-deployment.yaml
templates/web-service.yaml
templates/web-configmap.yaml

# ❌ 避免的命名
# 避免使用下划线或驼峰命名
name: web_app
name: webApp
```

#### 3. 模板组织

```yaml
# ✅ 模板最佳实践
# 1. 使用 _helpers.tpl 定义通用函数
{{- define "webapp.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

# 2. 使用一致的标签
metadata:
  labels:
    {{- include "webapp.labels" . | nindent 4 }}

# 3. 提供合理的默认值
image:
  repository: {{ .Values.image.repository }}
  tag: {{ .Values.image.tag | default .Chart.AppVersion }}

# 4. 使用条件渲染
{{- if .Values.ingress.enabled }}
# ingress 配置
{{- end }}
```

#### 4. Values 设计

```yaml
# ✅ values.yaml 最佳实践
# 1. 分层组织
global:
  imageRegistry: ""
  
webapp:
  image:
    repository: nginx
    tag: "1.20"
  service:
    type: ClusterIP
    port: 80

# 2. 提供示例和注释
ingress:
  enabled: false
  # className: nginx
  annotations: {}
    # kubernetes.io/ingress.class: nginx
    # cert-manager.io/cluster-issuer: letsencrypt
  hosts:
    - host: chart-example.local  # 示例域名
      paths:
        - path: /
          pathType: Prefix

# 3. 使用合理的默认值
resources:
  limits:
    cpu: 500m      # 适中的 CPU 限制
    memory: 512Mi  # 适中的内存限制
  requests:
    cpu: 250m      # 保守的请求值
    memory: 256Mi
```

---

## Helm 内置对象 (Builtin Objects)

### 什么是 Helm 内置对象？

Helm 内置对象是模板引擎提供的预定义变量，包含了 Chart、Release、Kubernetes 集群等相关信息。这些对象在模板渲染时自动可用，无需额外定义。

### 为什么需要了解内置对象？

1. **动态配置**：根据运行时信息动态生成资源配置
2. **环境适配**：基于不同环境和发布信息调整行为
3. **元数据管理**：自动生成标签、注解等元数据
4. **条件渲染**：基于对象属性实现智能模板逻辑

### 核心内置对象详解

#### 1. .Values 对象

`.Values` 包含了从 `values.yaml` 文件、用户提供的 values 文件和 `--set` 参数传入的所有值。

```yaml
# values.yaml
image:
  repository: nginx
  tag: "1.20"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

config:
  debug: false
  database:
    host: localhost
    port: 5432
```

**在模板中使用：**
```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "myapp.fullname" . }}
spec:
  template:
    spec:
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        imagePullPolicy: {{ .Values.image.pullPolicy }}
        ports:
        - containerPort: {{ .Values.service.port }}
        env:
        - name: DEBUG
          value: {{ .Values.config.debug | quote }}
        - name: DB_HOST
          value: {{ .Values.config.database.host | quote }}
        - name: DB_PORT
          value: {{ .Values.config.database.port | quote }}
```

**高级用法：**
```yaml
# 条件渲染
{{- if .Values.config.debug }}
        - name: LOG_LEVEL
          value: "debug"
{{- end }}

# 默认值处理
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default "latest" }}"

# 复杂数据结构遍历
{{- range $key, $value := .Values.config.database }}
        - name: {{ $key | upper }}
          value: {{ $value | quote }}
{{- end }}
```

#### 2. .Chart 对象

`.Chart` 包含了 `Chart.yaml` 文件中的所有信息。

```yaml
# Chart.yaml
apiVersion: v2
name: webapp
description: A Helm chart for web application
type: application
version: 0.1.0
appVersion: "1.16.0"
home: https://example.com
keywords:
  - web
  - nginx
maintainers:
  - name: John Doe
    email: john@example.com
```

**在模板中使用：**
```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
    app.kubernetes.io/name: {{ .Chart.Name }}
    app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
    app.kubernetes.io/managed-by: {{ .Release.Service }}
    helm.sh/chart: {{ .Chart.Name }}-{{ .Chart.Version | replace "+" "_" }}
  annotations:
    helm.sh/chart-description: {{ .Chart.Description | quote }}
    helm.sh/chart-home: {{ .Chart.Home | quote }}
spec:
  template:
    metadata:
      labels:
        app: {{ .Chart.Name }}
        version: {{ .Chart.AppVersion }}
    spec:
      containers:
      - name: {{ .Chart.Name }}
        # 使用 Chart 版本作为镜像标签
        image: "nginx:{{ .Chart.AppVersion }}"
```

**实用示例：**
```yaml
# 生成 Chart 相关的 ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "myapp.fullname" . }}-info
data:
  chart-name: {{ .Chart.Name | quote }}
  chart-version: {{ .Chart.Version | quote }}
  app-version: {{ .Chart.AppVersion | quote }}
  description: {{ .Chart.Description | quote }}
  keywords: {{ .Chart.Keywords | join "," | quote }}
  maintainers: {{ range .Chart.Maintainers }}{{ .name }}<{{ .email }}>{{ end }}
```

#### 3. .Release 对象

`.Release` 包含了当前发布的相关信息。

**可用属性：**
- `.Release.Name` - Release 名称
- `.Release.Namespace` - 发布的命名空间
- `.Release.Service` - 发布服务（通常是 "Helm"）
- `.Release.Revision` - 发布版本号
- `.Release.IsUpgrade` - 是否为升级操作
- `.Release.IsInstall` - 是否为安装操作

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-webapp
  namespace: {{ .Release.Namespace }}
  labels:
    app: {{ .Release.Name }}
    release: {{ .Release.Name }}
    heritage: {{ .Release.Service }}
  annotations:
    deployment.kubernetes.io/revision: {{ .Release.Revision | quote }}
spec:
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}
        pod-template-hash: {{ .Release.Name }}-{{ .Release.Revision }}
    spec:
      containers:
      - name: webapp
        env:
        - name: RELEASE_NAME
          value: {{ .Release.Name | quote }}
        - name: NAMESPACE
          value: {{ .Release.Namespace | quote }}
        - name: REVISION
          value: {{ .Release.Revision | quote }}
```

**高级用法：**
```yaml
# 根据操作类型执行不同逻辑
{{- if .Release.IsInstall }}
# 仅在安装时创建的资源
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ .Release.Name }}-install-job
spec:
  template:
    spec:
      containers:
      - name: installer
        image: installer:latest
        command: ["sh", "-c", "echo 'Installing {{ .Release.Name }}'"]
{{- end }}

{{- if .Release.IsUpgrade }}
# 仅在升级时创建的资源
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ .Release.Name }}-upgrade-job-{{ .Release.Revision }}
spec:
  template:
    spec:
      containers:
      - name: upgrader
        image: upgrader:latest
        command: ["sh", "-c", "echo 'Upgrading {{ .Release.Name }} to revision {{ .Release.Revision }}'"]
{{- end }}
```

#### 4. .Template 对象

`.Template` 包含了当前模板文件的信息。

**可用属性：**
- `.Template.Name` - 模板文件路径
- `.Template.BasePath` - 模板基础路径

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "myapp.fullname" . }}
  annotations:
    # 记录生成此资源的模板
    helm.sh/generated-by: {{ .Template.Name }}
    helm.sh/template-basepath: {{ .Template.BasePath }}
spec:
  template:
    metadata:
      annotations:
        # 可用于调试和审计
        config-source: "Generated from {{ .Template.Name }}"
```

#### 5. .Files 对象

`.Files` 提供对 Chart 中非模板文件的访问。

**Chart 文件结构：**
```
mychart/
├── Chart.yaml
├── values.yaml
├── templates/
│   └── deployment.yaml
├── config/
│   ├── app.conf
│   └── database.yaml
└── scripts/
    ├── init.sh
    └── backup.sh
```

**可用方法：**
- `.Files.Get` - 获取文件内容
- `.Files.GetBytes` - 获取文件字节内容
- `.Files.Glob` - 通过模式匹配文件
- `.Files.Lines` - 按行获取文件内容
- `.Files.AsSecrets` - 将文件内容转换为 base64（用于 Secret）
- `.Files.AsConfig` - 将文件内容用于 ConfigMap

```yaml
# config/app.conf
server.port=8080
server.host=0.0.0.0
database.url=jdbc:postgresql://localhost:5432/mydb

# config/database.yaml
host: localhost
port: 5432
username: admin
ssl: true
```

**在模板中使用：**
```yaml
# templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "myapp.fullname" . }}-config
data:
  # 直接包含文件内容
  app.conf: |
{{ .Files.Get "config/app.conf" | indent 4 }}
  
  # 包含 YAML 文件
  database.yaml: |
{{ .Files.Get "config/database.yaml" | indent 4 }}
  
  # 动态处理多个配置文件
{{- range $path, $_ := .Files.Glob "config/*.conf" }}
  {{ base $path }}: |
{{ $.Files.Get $path | indent 4 }}
{{- end }}
```

**创建 Secret：**
```yaml
# templates/secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: {{ include "myapp.fullname" . }}-files
type: Opaque
data:
{{- range $path, $_ := .Files.Glob "scripts/*" }}
  {{ base $path }}: {{ $.Files.Get $path | b64enc }}
{{- end }}

# 或使用便捷方法
# data:
# {{ (.Files.Glob "scripts/*").AsSecrets | indent 2 }}
```

**高级文件处理：**
```yaml
# templates/configmap-advanced.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "myapp.fullname" . }}-scripts
data:
{{- range $path, $_ := .Files.Glob "scripts/*.sh" }}
  {{ base $path }}: |
    #!/bin/bash
    # Generated from {{ $path }}
    # Release: {{ $.Release.Name }}
    # Namespace: {{ $.Release.Namespace }}
    
{{ $.Files.Get $path | indent 4 }}
{{- end }}
```

#### 6. .Capabilities 对象

`.Capabilities` 包含了 Kubernetes 集群的能力信息。

**可用属性：**
- `.Capabilities.KubeVersion` - Kubernetes 版本信息
- `.Capabilities.APIVersions` - 可用的 API 版本
- `.Capabilities.HelmVersion` - Helm 版本信息

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "myapp.fullname" . }}
  annotations:
    kubernetes-version: {{ .Capabilities.KubeVersion }}
    helm-version: {{ .Capabilities.HelmVersion }}
spec:
  template:
    spec:
      containers:
      - name: webapp
        env:
        - name: KUBE_VERSION
          value: {{ .Capabilities.KubeVersion | quote }}
```

**版本兼容性检查：**
```yaml
# templates/ingress.yaml
{{- if .Values.ingress.enabled -}}
{{- if semverCompare ">=1.19-0" .Capabilities.KubeVersion.GitVersion -}}
apiVersion: networking.k8s.io/v1
{{- else if semverCompare ">=1.14-0" .Capabilities.KubeVersion.GitVersion -}}
apiVersion: networking.k8s.io/v1beta1
{{- else -}}
apiVersion: extensions/v1beta1
{{- end }}
kind: Ingress
metadata:
  name: {{ include "myapp.fullname" . }}
spec:
  # 配置内容...
{{- end }}
```

**API 版本检查：**
```yaml
# templates/poddisruptionbudget.yaml
{{- if .Values.podDisruptionBudget.enabled }}
{{- if .Capabilities.APIVersions.Has "policy/v1/PodDisruptionBudget" }}
apiVersion: policy/v1
{{- else }}
apiVersion: policy/v1beta1
{{- end }}
kind: PodDisruptionBudget
metadata:
  name: {{ include "myapp.fullname" . }}
spec:
  minAvailable: {{ .Values.podDisruptionBudget.minAvailable }}
  selector:
    matchLabels:
      {{- include "myapp.selectorLabels" . | nindent 6 }}
{{- end }}
```

### 内置对象的高级应用

#### 1. 组合使用多个对象

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-{{ .Chart.Name }}
  labels:
    # 组合 Chart 和 Release 信息
    app.kubernetes.io/name: {{ .Chart.Name }}
    app.kubernetes.io/instance: {{ .Release.Name }}
    app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
    app.kubernetes.io/managed-by: {{ .Release.Service }}
    helm.sh/chart: {{ .Chart.Name }}-{{ .Chart.Version | replace "+" "_" }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app.kubernetes.io/name: {{ .Chart.Name }}
      app.kubernetes.io/instance: {{ .Release.Name }}
  template:
    metadata:
      labels:
        app.kubernetes.io/name: {{ .Chart.Name }}
        app.kubernetes.io/instance: {{ .Release.Name }}
      annotations:
        # 使用 Template 对象进行调试
        helm.sh/template-source: {{ .Template.Name }}
        # 结合 Files 对象计算配置哈希
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
    spec:
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
        env:
        - name: RELEASE_INFO
          value: "{{ .Release.Name }}/{{ .Release.Namespace }}/{{ .Release.Revision }}"
        - name: CHART_INFO
          value: "{{ .Chart.Name }}/{{ .Chart.Version }}"
```

#### 2. 动态资源命名

```yaml
# templates/_helpers.tpl
{{/*
生成完整的应用名称
*/}}
{{- define "myapp.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- if contains $name .Release.Name }}
{{- .Release.Name | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
{{- end }}

{{/*
生成通用标签
*/}}
{{- define "myapp.labels" -}}
helm.sh/chart: {{ include "myapp.chart" . }}
{{ include "myapp.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
release-revision: {{ .Release.Revision | quote }}
{{- end }}
```

#### 3. 条件资源创建

```yaml
# templates/job-migration.yaml
{{- if and .Release.IsInstall .Values.database.migrate }}
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "myapp.fullname" . }}-migration
  annotations:
    "helm.sh/hook": pre-install
    "helm.sh/hook-weight": "-5"
    "helm.sh/hook-delete-policy": hook-succeeded
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: migration
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
        command: ["migrate"]
        env:
        - name: DATABASE_URL
          value: {{ .Values.database.url | quote }}
{{- end }}

# templates/job-upgrade.yaml
{{- if and .Release.IsUpgrade .Values.maintenance.enabled }}
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "myapp.fullname" . }}-maintenance-{{ .Release.Revision }}
  annotations:
    "helm.sh/hook": pre-upgrade
    "helm.sh/hook-weight": "-10"
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: maintenance
        image: maintenance:latest
        env:
        - name: RELEASE_NAME
          value: {{ .Release.Name | quote }}
        - name: OLD_REVISION
          value: {{ sub .Release.Revision 1 | quote }}
        - name: NEW_REVISION
          value: {{ .Release.Revision | quote }}
{{- end }}
```

#### 4. 文件内容动态处理

```yaml
# templates/configmap-dynamic.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "myapp.fullname" . }}-config
data:
  # 处理配置文件模板
{{- range $path, $_ := .Files.Glob "config/*.tmpl" }}
  {{- $filename := base $path | trimSuffix ".tmpl" }}
  {{ $filename }}: |
{{- tpl ($.Files.Get $path) $ | indent 4 }}
{{- end }}
  
  # 生成应用信息文件
  app-info.json: |
    {
      "name": {{ .Chart.Name | quote }},
      "version": {{ .Chart.AppVersion | quote }},
      "release": {{ .Release.Name | quote }},
      "namespace": {{ .Release.Namespace | quote }},
      "revision": {{ .Release.Revision }},
      "kubeVersion": {{ .Capabilities.KubeVersion | quote }}
    }
```

### 内置对象最佳实践

#### 1. 防御性编程

```yaml
# 安全地访问可能不存在的值
{{- if and .Values.database .Values.database.enabled }}
- name: DATABASE_URL
  value: {{ .Values.database.url | default "postgresql://localhost:5432/app" | quote }}
{{- end }}

# 使用 hasKey 检查键是否存在
{{- if hasKey .Values "monitoring" }}
{{- if .Values.monitoring.enabled }}
# 监控配置
{{- end }}
{{- end }}

# 类型检查和默认值
replicas: {{ .Values.replicaCount | default 1 | int }}
```

#### 2. 标准化元数据

```yaml
# templates/_helpers.tpl
{{/*
标准 Kubernetes 标签
*/}}
{{- define "myapp.labels" -}}
app.kubernetes.io/name: {{ include "myapp.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
app.kubernetes.io/part-of: {{ .Chart.Name }}
helm.sh/chart: {{ include "myapp.chart" . }}
{{- end }}

{{/*
标准注解
*/}}
{{- define "myapp.annotations" -}}
meta.helm.sh/release-name: {{ .Release.Name }}
meta.helm.sh/release-namespace: {{ .Release.Namespace }}
{{- if .Chart.AppVersion }}
app.version: {{ .Chart.AppVersion | quote }}
{{- end }}
{{- end }}
```

#### 3. 版本兼容性管理

```yaml
# templates/networkpolicy.yaml
{{- if .Values.networkPolicy.enabled }}
{{- if .Capabilities.APIVersions.Has "networking.k8s.io/v1/NetworkPolicy" }}
apiVersion: networking.k8s.io/v1
{{- else }}
{{- fail "NetworkPolicy requires Kubernetes 1.7+" }}
{{- end }}
kind: NetworkPolicy
metadata:
  name: {{ include "myapp.fullname" . }}
spec:
  # 配置内容...
{{- end }}
```

#### 4. 调试和审计支持

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "myapp.fullname" . }}
  annotations:
    # 调试信息
    helm.sh/chart-path: {{ .Template.BasePath }}
    helm.sh/template-name: {{ .Template.Name }}
    helm.sh/values-checksum: {{ .Values | toYaml | sha256sum }}
    # 审计信息
    deployment.revision: {{ .Release.Revision | quote }}
    deployment.timestamp: {{ now | unixEpoch | quote }}
spec:
  template:
    metadata:
      annotations:
        # Pod 级别的调试信息
        kubectl.kubernetes.io/restartedAt: {{ now | quote }}
        config.hash: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
```

---

## Helm 开发基础 (Development Basics)

### 什么是 Helm 模板开发？

Helm 模板开发是使用 Go 模板语言创建动态 Kubernetes 资源定义的过程。通过模板语法，您可以创建可配置、可重用的 Chart，支持不同环境和需求。

### 为什么需要掌握模板开发基础？

1. **动态配置**：根据不同环境生成不同的资源配置
2. **代码复用**：通过模板减少重复配置
3. **条件逻辑**：实现智能的资源创建和配置
4. **数据处理**：格式化和转换配置数据

### Step-01: Helm Template Actions and Action Elements

#### 什么是 Template Actions？

Template Actions 是 Helm 模板中的动态元素，用双花括号 `{{ }}` 包围。它们在模板渲染时被执行并替换为实际值。

#### Action Elements 类型

**1. 变量输出 (Variable Output)**
```yaml
# 基本变量输出
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-webapp
  namespace: {{ .Release.Namespace }}
  labels:
    app: {{ .Chart.Name }}
    version: {{ .Chart.AppVersion }}
```

**2. 函数调用 (Function Calls)**
```yaml
# 使用内置函数
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
    app.kubernetes.io/name: {{ include "myapp.name" . }}
    app.kubernetes.io/instance: {{ .Release.Name }}
    app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
```

**3. 管道操作 (Pipeline Operations)**
```yaml
# 链式函数调用
spec:
  replicas: {{ .Values.replicaCount | default 1 | int }}
  containers:
  - name: {{ .Chart.Name | lower }}
    image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
    port: {{ .Values.service.port | toString | quote }}
```

**4. 条件语句 (Conditional Statements)**
```yaml
# if-else 条件
spec:
  {{- if .Values.autoscaling.enabled }}
  replicas: {{ .Values.autoscaling.minReplicas }}
  {{- else }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  
  containers:
  - name: webapp
    {{- if .Values.image.tag }}
    image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
    {{- else }}
    image: "{{ .Values.image.repository }}:{{ .Chart.AppVersion }}"
    {{- end }}
```

**5. 循环语句 (Loop Statements)**
```yaml
# range 循环
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "myapp.fullname" . }}-config
data:
{{- range $key, $value := .Values.config }}
  {{ $key }}: {{ $value | quote }}
{{- end }}

# 数组循环
{{- range .Values.ingress.hosts }}
  - host: {{ .host | quote }}
    http:
      paths:
      {{- range .paths }}
      - path: {{ .path }}
        pathType: {{ .pathType }}
        backend:
          service:
            name: {{ include "myapp.fullname" $ }}
            port:
              number: {{ $.Values.service.port }}
      {{- end }}
{{- end }}
```

#### Invalid Action Elements 示例

**常见错误和修正：**

```yaml
# ❌ 错误：缺少空格
{{.Release.Name}}

# ✅ 正确：Action 内部需要空格
{{ .Release.Name }}

# ❌ 错误：无效的变量引用
{{ .InvalidObject.Property }}

# ✅ 正确：使用 default 处理可能不存在的值
{{ .Values.optional.property | default "defaultValue" }}

# ❌ 错误：在字符串中间使用 Action
name: prefix-{{ .Release.Name }}-suffix-{{ .Values.suffix }}

# ✅ 正确：合并为单个 Action
name: {{ printf "prefix-%s-suffix-%s" .Release.Name .Values.suffix }}

# ❌ 错误：Action 语法不完整
{{- if .Values.enabled
  enabled: true
{{- end }}

# ✅ 正确：完整的 Action 语法
{{- if .Values.enabled }}
  enabled: true
{{- end }}
```

### Step-03: Helm Quote Function and Pipeline

#### Quote 函数详解

Quote 函数用于为值添加双引号，确保 YAML 格式正确，特别是处理包含特殊字符或可能被误解析的值。

**基本用法：**
```yaml
# 字符串值加引号
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "myapp.fullname" . }}
data:
  app-name: {{ .Chart.Name | quote }}
  app-version: {{ .Chart.AppVersion | quote }}
  release-name: {{ .Release.Name | quote }}
  namespace: {{ .Release.Namespace | quote }}
```

**何时使用 Quote：**
```yaml
# 1. 数字值作为字符串
data:
  port: {{ .Values.service.port | quote }}           # "80"
  version: {{ .Values.app.version | quote }}         # "1.0"
  
# 2. 布尔值作为字符串  
  debug: {{ .Values.debug | quote }}                 # "true"
  
# 3. 包含特殊字符的值
  database-url: {{ .Values.database.url | quote }}   # "postgresql://user:pass@host:5432/db"
  
# 4. 可能为空的值
  optional-field: {{ .Values.optional | default "" | quote }}
```

**实际应用示例：**
```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
spec:
  template:
    spec:
      containers:
      - name: {{ .Chart.Name }}
        env:
        - name: APP_NAME
          value: {{ .Chart.Name | quote }}
        - name: APP_VERSION  
          value: {{ .Chart.AppVersion | quote }}
        - name: RELEASE_NAME
          value: {{ .Release.Name | quote }}
        - name: LOG_LEVEL
          value: {{ .Values.logLevel | default "info" | quote }}
        - name: DATABASE_PORT
          value: {{ .Values.database.port | toString | quote }}
        - name: ENABLE_DEBUG
          value: {{ .Values.debug | quote }}
```

#### Pipeline 详解

Pipeline 是 Helm 模板中的强大特性，允许将值通过一系列函数进行处理，类似 Unix 的管道操作。

**Pipeline 语法：**
```yaml
# 基本语法：value | function1 | function2 | function3
{{ .Values.input | function1 | function2 | function3 }}
```

**常用 Pipeline 组合：**
```yaml
# 1. 字符串处理 Pipeline
apiVersion: v1
kind: Service
metadata:
  name: {{ .Release.Name | lower | trunc 63 | trimSuffix "-" }}
  labels:
    app: {{ .Chart.Name | lower }}
    version: {{ .Chart.AppVersion | replace "." "-" }}

# 2. 数值处理 Pipeline  
spec:
  replicas: {{ .Values.replicaCount | default 1 | int }}
  ports:
  - port: {{ .Values.service.port | default 80 | int }}
    targetPort: {{ .Values.service.targetPort | default .Values.service.port | int }}

# 3. 条件和默认值 Pipeline
data:
  config.yaml: |
    app:
      name: {{ .Chart.Name | default "myapp" | quote }}
      debug: {{ .Values.debug | default false }}
      workers: {{ .Values.workers | default 4 | int }}
      timeout: {{ .Values.timeout | default "30s" | quote }}
```

**复杂 Pipeline 示例：**
```yaml
# templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "myapp.fullname" . }}-config
data:
  # 字符串清理和格式化
  clean-name: {{ .Values.appName | lower | replace " " "-" | trunc 63 | trimSuffix "-" | quote }}
  
  # URL 构建
  database-url: {{ printf "postgresql://%s:%s@%s:%d/%s" 
    .Values.database.username 
    .Values.database.password 
    .Values.database.host 
    (.Values.database.port | int) 
    .Values.database.name | quote }}
  
  # 复杂条件处理
  log-level: {{ .Values.logLevel | default (ternary "debug" "info" .Values.debug) | upper | quote }}
  
  # 列表处理
  allowed-hosts: {{ .Values.allowedHosts | default (list "localhost") | join "," | quote }}
```

### Step-04: Helm Default Function

#### Default 函数详解

Default 函数用于为可能为空或未定义的值提供默认值，是 Helm 模板中最常用的函数之一。

**基本语法：**
```yaml
{{ .Values.someValue | default "defaultValue" }}
```

**使用场景：**

**1. 简单默认值：**
```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "myapp.fullname" . }}
spec:
  replicas: {{ .Values.replicaCount | default 1 }}
  template:
    spec:
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
        imagePullPolicy: {{ .Values.image.pullPolicy | default "IfNotPresent" }}
        ports:
        - containerPort: {{ .Values.service.targetPort | default 8080 }}
```

**2. 嵌套默认值：**
```yaml
# templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "myapp.fullname" . }}
  annotations:
    {{- if .Values.service.annotations }}
    {{- toYaml .Values.service.annotations | nindent 4 }}
    {{- end }}
spec:
  type: {{ .Values.service.type | default "ClusterIP" }}
  ports:
  - port: {{ .Values.service.port | default 80 }}
    targetPort: {{ .Values.service.targetPort | default .Values.service.port | default 8080 }}
    protocol: {{ .Values.service.protocol | default "TCP" }}
    name: {{ .Values.service.name | default "http" }}
```

**3. 复杂对象默认值：**
```yaml
# templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "myapp.fullname" . }}-config
data:
  # 数据库配置默认值
  db-host: {{ .Values.database.host | default "localhost" | quote }}
  db-port: {{ .Values.database.port | default 5432 | quote }}
  db-name: {{ .Values.database.name | default .Chart.Name | quote }}
  db-ssl: {{ .Values.database.ssl | default true | quote }}
  
  # 应用配置默认值
  log-level: {{ .Values.logging.level | default "info" | quote }}
  log-format: {{ .Values.logging.format | default "json" | quote }}
  max-connections: {{ .Values.app.maxConnections | default 100 | quote }}
  timeout: {{ .Values.app.timeout | default "30s" | quote }}
```

**4. 条件默认值：**
```yaml
# templates/deployment.yaml
spec:
  template:
    spec:
      containers:
      - name: {{ .Chart.Name }}
        env:
        - name: ENVIRONMENT
          value: {{ .Values.environment | default (ternary "production" "development" .Release.IsInstall) | quote }}
        - name: LOG_LEVEL
          value: {{ .Values.logLevel | default (ternary "debug" "info" .Values.debug) | quote }}
        - name: WORKERS
          value: {{ .Values.workers | default (ternary "1" "4" (eq .Values.environment "development")) | quote }}
```

### Step-05: Controlling Leading and Trailing Whitespaces

#### 空白字符控制

Helm 模板中的空白字符控制对于生成格式正确的 YAML 至关重要。Go 模板提供了特殊语法来控制前导和尾随空白。

**基本语法：**
- `{{-` : 删除前导空白
- `-}}` : 删除尾随空白  
- `{{- -}}` : 删除前导和尾随空白

**问题演示：**
```yaml
# ❌ 不当的空白处理
apiVersion: v1
kind: ConfigMap
metadata:
  name: example
data:
{{ if .Values.config.enabled }}
  enabled: "true"
{{ end }}
  always-present: "value"

# 生成的 YAML（注意多余的空行）:
# data:
#
#   enabled: "true"
#
#   always-present: "value"
```

**正确的空白控制：**
```yaml
# ✅ 正确的空白处理
apiVersion: v1
kind: ConfigMap
metadata:
  name: example
data:
  {{- if .Values.config.enabled }}
  enabled: "true"
  {{- end }}
  always-present: "value"

# 生成的 YAML（干净格式）:
# data:
#   enabled: "true"
#   always-present: "value"
```

**实际应用示例：**

**1. 条件块的空白控制：**
```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
  {{- if .Values.annotations }}
  annotations:
    {{- toYaml .Values.annotations | nindent 4 }}
  {{- end }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "myapp.selectorLabels" . | nindent 6 }}
```

**2. 循环中的空白控制：**
```yaml
# templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "myapp.fullname" . }}-config
data:
  {{- range $key, $value := .Values.config }}
  {{ $key }}: {{ $value | quote }}
  {{- end }}
  {{- if .Values.extraConfig }}
  {{- range $key, $value := .Values.extraConfig }}
  {{ $key }}: {{ $value | quote }}
  {{- end }}
  {{- end }}
```

**3. 复杂嵌套的空白控制：**
```yaml
# templates/ingress.yaml
{{- if .Values.ingress.enabled -}}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
  {{- with .Values.ingress.annotations }}
  annotations:
    {{- toYaml . | nindent 4 }}
  {{- end }}
spec:
  {{- if .Values.ingress.className }}
  ingressClassName: {{ .Values.ingress.className }}
  {{- end }}
  {{- if .Values.ingress.tls }}
  tls:
    {{- range .Values.ingress.tls }}
    - hosts:
        {{- range .hosts }}
        - {{ . | quote }}
        {{- end }}
      secretName: {{ .secretName }}
    {{- end }}
  {{- end }}
  rules:
    {{- range .Values.ingress.hosts }}
    - host: {{ .host | quote }}
      http:
        paths:
          {{- range .paths }}
          - path: {{ .path }}
            pathType: {{ .pathType }}
            backend:
              service:
                name: {{ include "myapp.fullname" $ }}
                port:
                  number: {{ $.Values.service.port }}
          {{- end }}
    {{- end }}
{{- end }}
```

### Step-06: Indent and Nindent Functions

#### Indent 和 Nindent 函数详解

这两个函数用于控制 YAML 的缩进，确保生成的配置文件格式正确。

**函数区别：**
- `indent N` : 在每行前添加 N 个空格
- `nindent N` : 先添加换行符，然后在每行前添加 N 个空格

**基本用法：**
```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
{{ include "myapp.labels" . | indent 4 }}
  # 或者使用 nindent（推荐）
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
```

**实际应用示例：**

**1. 标签和注解的缩进：**
```yaml
# templates/_helpers.tpl
{{- define "myapp.labels" -}}
app.kubernetes.io/name: {{ include "myapp.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
  {{- if .Values.annotations }}
  annotations:
    {{- toYaml .Values.annotations | nindent 4 }}
  {{- end }}
```

**2. 复杂配置的缩进：**
```yaml
# templates/deployment.yaml
spec:
  template:
    spec:
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
      containers:
      - name: {{ .Chart.Name }}
        {{- if .Values.securityContext }}
        securityContext:
          {{- toYaml .Values.securityContext | nindent 10 }}
        {{- end }}
        {{- if .Values.resources }}
        resources:
          {{- toYaml .Values.resources | nindent 10 }}
        {{- end }}
```

**3. 环境变量的缩进：**
```yaml
# templates/deployment.yaml
containers:
- name: {{ .Chart.Name }}
  env:
  - name: APP_NAME
    value: {{ .Chart.Name | quote }}
  {{- if .Values.extraEnv }}
  {{- range $key, $value := .Values.extraEnv }}
  - name: {{ $key }}
    value: {{ $value | quote }}
  {{- end }}
  {{- end }}
  {{- if .Values.envFrom }}
  envFrom:
    {{- toYaml .Values.envFrom | nindent 2 }}
  {{- end }}
```

### Step-07: toYaml Function

#### toYaml 函数详解

toYaml 函数将 Helm 值转换为 YAML 格式的字符串，常用于处理复杂的数据结构。

**基本用法：**
```yaml
# values.yaml
resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 250m
    memory: 256Mi

nodeSelector:
  kubernetes.io/os: linux
  node-type: worker

# templates/deployment.yaml
spec:
  template:
    spec:
      {{- with .Values.nodeSelector }}
      nodeSelector:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      containers:
      - name: {{ .Chart.Name }}
        {{- if .Values.resources }}
        resources:
          {{- toYaml .Values.resources | nindent 10 }}
        {{- end }}
```

**高级应用：**

**1. 复杂配置结构：**
```yaml
# values.yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/e2e-az-name
          operator: In
          values:
          - e2e-az1
          - e2e-az2

tolerations:
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoSchedule"
- key: "key2"
  operator: "Exists"
  effect: "NoExecute"

# templates/deployment.yaml
spec:
  template:
    spec:
      {{- with .Values.affinity }}
      affinity:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.tolerations }}
      tolerations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
```

**2. 条件 toYaml 使用：**
```yaml
# templates/deployment.yaml
spec:
  template:
    spec:
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
        {{- if .Values.command }}
        command:
          {{- toYaml .Values.command | nindent 10 }}
        {{- end }}
        {{- if .Values.args }}
        args:
          {{- toYaml .Values.args | nindent 10 }}
        {{- end }}
        {{- if .Values.env }}
        env:
          {{- toYaml .Values.env | nindent 10 }}
        {{- end }}
        {{- if .Values.volumeMounts }}
        volumeMounts:
          {{- toYaml .Values.volumeMounts | nindent 10 }}
        {{- end }}
      {{- if .Values.volumes }}
      volumes:
        {{- toYaml .Values.volumes | nindent 8 }}
      {{- end }}
```

**3. toYaml 与其他函数组合：**
```yaml
# templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "myapp.fullname" . }}-config
data:
  config.yaml: |
    app:
      name: {{ .Chart.Name | quote }}
      version: {{ .Chart.AppVersion | quote }}
    {{- if .Values.appConfig }}
    {{- toYaml .Values.appConfig | nindent 4 }}
    {{- end }}
  
  # 使用 toYaml 处理复杂配置
  database.yaml: |
    {{- if .Values.database }}
    {{- toYaml .Values.database | nindent 4 }}
    {{- else }}
    host: localhost
    port: 5432
    name: {{ .Chart.Name }}
    {{- end }}
```

### 开发基础综合示例

结合所有概念的完整示例：

```yaml
# templates/deployment.yaml
{{- $fullName := include "myapp.fullname" . -}}
{{- $labels := include "myapp.labels" . -}}
{{- $selectorLabels := include "myapp.selectorLabels" . -}}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ $fullName }}
  labels:
    {{- $labels | nindent 4 }}
  {{- with .Values.annotations }}
  annotations:
    {{- toYaml . | nindent 4 }}
  {{- end }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount | default 1 }}
  {{- end }}
  selector:
    matchLabels:
      {{- $selectorLabels | nindent 6 }}
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
        {{- with .Values.podAnnotations }}
        {{- toYaml . | nindent 8 }}
        {{- end }}
      labels:
        {{- $selectorLabels | nindent 8 }}
    spec:
      {{- with .Values.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      serviceAccountName: {{ include "myapp.serviceAccountName" . }}
      {{- with .Values.podSecurityContext }}
      securityContext:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      containers:
      - name: {{ .Chart.Name }}
        {{- with .Values.securityContext }}
        securityContext:
          {{- toYaml . | nindent 10 }}
        {{- end }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
        imagePullPolicy: {{ .Values.image.pullPolicy | default "IfNotPresent" }}
        ports:
        - name: http
          containerPort: {{ .Values.service.targetPort | default 8080 }}
          protocol: TCP
        {{- if .Values.livenessProbe.enabled | default true }}
        livenessProbe:
          httpGet:
            path: {{ .Values.livenessProbe.path | default "/health" }}
            port: http
          initialDelaySeconds: {{ .Values.livenessProbe.initialDelaySeconds | default 30 }}
          periodSeconds: {{ .Values.livenessProbe.periodSeconds | default 10 }}
        {{- end }}
        {{- if .Values.readinessProbe.enabled | default true }}
        readinessProbe:
          httpGet:
            path: {{ .Values.readinessProbe.path | default "/ready" }}
            port: http
          initialDelaySeconds: {{ .Values.readinessProbe.initialDelaySeconds | default 5 }}
          periodSeconds: {{ .Values.readinessProbe.periodSeconds | default 5 }}
        {{- end }}
        env:
        - name: APP_NAME
          value: {{ .Chart.Name | quote }}
        - name: APP_VERSION
          value: {{ .Chart.AppVersion | quote }}
        - name: RELEASE_NAME
          value: {{ .Release.Name | quote }}
        {{- if .Values.extraEnv }}
        {{- range $key, $value := .Values.extraEnv }}
        - name: {{ $key | upper }}
          value: {{ $value | quote }}
        {{- end }}
        {{- end }}
        {{- with .Values.resources }}
        resources:
          {{- toYaml . | nindent 10 }}
        {{- end }}
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

---

## Helm 高级控制结构 (Advanced Control Structures)

### 什么是高级控制结构？

Helm 高级控制结构提供了复杂的条件逻辑、流程控制和数据处理能力。这些功能使得 Chart 能够根据不同的配置、环境和条件动态生成适合的 Kubernetes 资源。

### 为什么需要高级控制结构？

1. **智能配置**：根据复杂条件决定资源的创建和配置
2. **环境适配**：支持多环境部署的不同需求
3. **数据处理**：高效处理列表、字典等复杂数据结构
4. **代码复用**：通过变量和循环减少重复代码

### Demo-13: Helm If, Else with EQ Function

#### EQ 函数详解

EQ (Equal) 函数用于比较两个值是否相等，是 Helm 模板中最常用的比较函数之一。

**基本语法：**
```yaml
{{ eq .Values.value1 .Values.value2 }}
{{ if eq .Values.environment "production" }}
```

**实际应用示例：**

**1. 环境判断：**
```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "myapp.fullname" . }}
spec:
  {{- if eq .Values.environment "production" }}
  replicas: {{ .Values.replicaCount | default 3 }}
  {{- else if eq .Values.environment "staging" }}
  replicas: {{ .Values.replicaCount | default 2 }}
  {{- else }}
  replicas: {{ .Values.replicaCount | default 1 }}
  {{- end }}
  template:
    spec:
      containers:
      - name: {{ .Chart.Name }}
        {{- if eq .Values.environment "production" }}
        resources:
          limits:
            cpu: "1000m"
            memory: "1Gi"
          requests:
            cpu: "500m"
            memory: "512Mi"
        {{- else if eq .Values.environment "staging" }}
        resources:
          limits:
            cpu: "500m"
            memory: "512Mi"
          requests:
            cpu: "250m"
            memory: "256Mi"
        {{- else }}
        resources:
          limits:
            cpu: "200m"
            memory: "256Mi"
          requests:
            cpu: "100m"
            memory: "128Mi"
        {{- end }}
```

**2. 服务类型判断：**
```yaml
# templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "myapp.fullname" . }}
  {{- if eq .Values.service.type "LoadBalancer" }}
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
  {{- else if eq .Values.service.type "NodePort" }}
  annotations:
    service.kubernetes.io/external-traffic: "OnlyLocal"
  {{- end }}
spec:
  type: {{ .Values.service.type }}
  {{- if eq .Values.service.type "ClusterIP" }}
  clusterIP: {{ .Values.service.clusterIP | default "None" }}
  {{- else if eq .Values.service.type "LoadBalancer" }}
  loadBalancerSourceRanges:
  {{- range .Values.service.loadBalancerSourceRanges }}
  - {{ . }}
  {{- end }}
  {{- end }}
```

**3. 版本兼容性检查：**
```yaml
# templates/ingress.yaml
{{- if .Values.ingress.enabled }}
{{- if eq .Capabilities.KubeVersion.Major "1" }}
  {{- if eq .Capabilities.KubeVersion.Minor "19" }}
apiVersion: networking.k8s.io/v1
  {{- else if eq .Capabilities.KubeVersion.Minor "18" }}
apiVersion: networking.k8s.io/v1beta1
  {{- else }}
apiVersion: extensions/v1beta1
  {{- end }}
{{- end }}
kind: Ingress
metadata:
  name: {{ include "myapp.fullname" . }}
{{- end }}
```

### Demo-14: Helm If, Else with Boolean Check, AND Function

#### 布尔检查和 AND 函数

布尔检查用于验证值的真假，AND 函数用于组合多个条件，所有条件都为真时才执行。

**AND 函数语法：**
```yaml
{{ and .Values.condition1 .Values.condition2 .Values.condition3 }}
```

**实际应用示例：**

**1. 复合条件检查：**
```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "myapp.fullname" . }}
spec:
  template:
    spec:
      {{- if and .Values.persistence.enabled .Values.persistence.existingClaim }}
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: {{ .Values.persistence.existingClaim }}
      {{- else if and .Values.persistence.enabled (not .Values.persistence.existingClaim) }}
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: {{ include "myapp.fullname" . }}-data
      {{- end }}
      
      containers:
      - name: {{ .Chart.Name }}
        {{- if and .Values.persistence.enabled .Values.persistence.mountPath }}
        volumeMounts:
        - name: data
          mountPath: {{ .Values.persistence.mountPath }}
          {{- if .Values.persistence.subPath }}
          subPath: {{ .Values.persistence.subPath }}
          {{- end }}
        {{- end }}
```

**2. 安全配置检查：**
```yaml
# templates/deployment.yaml
spec:
  template:
    spec:
      containers:
      - name: {{ .Chart.Name }}
        {{- if and .Values.security.enabled .Values.security.runAsNonRoot }}
        securityContext:
          runAsNonRoot: true
          runAsUser: {{ .Values.security.runAsUser | default 1000 }}
          runAsGroup: {{ .Values.security.runAsGroup | default 1000 }}
          {{- if and .Values.security.capabilities .Values.security.capabilities.drop }}
          capabilities:
            drop:
            {{- range .Values.security.capabilities.drop }}
            - {{ . }}
            {{- end }}
          {{- end }}
        {{- end }}
```

**3. 监控和日志配置：**
```yaml
# templates/configmap.yaml
{{- if and .Values.monitoring.enabled .Values.logging.enabled }}
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "myapp.fullname" . }}-monitor-config
data:
  prometheus.yml: |
    global:
      scrape_interval: {{ .Values.monitoring.scrapeInterval | default "15s" }}
    scrape_configs:
    - job_name: '{{ .Chart.Name }}'
      static_configs:
      - targets: ['localhost:{{ .Values.service.port }}']
  
  fluent.conf: |
    <source>
      @type tail
      path {{ .Values.logging.path | default "/var/log/app.log" }}
      pos_file /var/log/fluentd-app.log.pos
      tag {{ .Chart.Name }}
      format json
    </source>
{{- end }}
```

### Demo-15: Helm If, Else with OR Function

#### OR 函数详解

OR 函数用于组合多个条件，只要其中一个条件为真就执行。

**OR 函数语法：**
```yaml
{{ or .Values.condition1 .Values.condition2 .Values.condition3 }}
```

**实际应用示例：**

**1. 多种存储选项：**
```yaml
# templates/deployment.yaml
{{- if or .Values.persistence.enabled .Values.emptyDir.enabled .Values.hostPath.enabled }}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "myapp.fullname" . }}
spec:
  template:
    spec:
      volumes:
      {{- if .Values.persistence.enabled }}
      - name: data
        persistentVolumeClaim:
          claimName: {{ include "myapp.fullname" . }}-data
      {{- else if .Values.emptyDir.enabled }}
      - name: data
        emptyDir:
          {{- if .Values.emptyDir.sizeLimit }}
          sizeLimit: {{ .Values.emptyDir.sizeLimit }}
          {{- end }}
      {{- else if .Values.hostPath.enabled }}
      - name: data
        hostPath:
          path: {{ .Values.hostPath.path }}
          type: {{ .Values.hostPath.type | default "DirectoryOrCreate" }}
      {{- end }}
{{- end }}
```

**2. 多种网络配置：**
```yaml
# templates/service.yaml
{{- if or (eq .Values.service.type "NodePort") (eq .Values.service.type "LoadBalancer") .Values.ingress.enabled }}
apiVersion: v1
kind: Service
metadata:
  name: {{ include "myapp.fullname" . }}
  {{- if or .Values.service.annotations .Values.global.annotations }}
  annotations:
    {{- if .Values.service.annotations }}
    {{- toYaml .Values.service.annotations | nindent 4 }}
    {{- end }}
    {{- if .Values.global.annotations }}
    {{- toYaml .Values.global.annotations | nindent 4 }}
    {{- end }}
  {{- end }}
spec:
  type: {{ .Values.service.type }}
  {{- if or (eq .Values.service.type "NodePort") (eq .Values.service.type "LoadBalancer") }}
  externalTrafficPolicy: {{ .Values.service.externalTrafficPolicy | default "Cluster" }}
  {{- end }}
{{- end }}
```

### Demo-16: Helm If, Else with NOT Function

#### NOT 函数详解

NOT 函数用于逻辑取反，当条件为假时返回真。

**NOT 函数语法：**
```yaml
{{ not .Values.condition }}
```

**实际应用示例：**

**1. 排除特定配置：**
```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "myapp.fullname" . }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount | default 1 }}
  {{- end }}
  template:
    spec:
      containers:
      - name: {{ .Chart.Name }}
        {{- if not .Values.development.enabled }}
        livenessProbe:
          httpGet:
            path: {{ .Values.livenessProbe.path | default "/health" }}
            port: http
          initialDelaySeconds: {{ .Values.livenessProbe.initialDelaySeconds | default 30 }}
        readinessProbe:
          httpGet:
            path: {{ .Values.readinessProbe.path | default "/ready" }}
            port: http
          initialDelaySeconds: {{ .Values.readinessProbe.initialDelaySeconds | default 5 }}
        {{- end }}
```

**2. 条件资源创建：**
```yaml
# templates/pdb.yaml
{{- if and (not .Values.development.enabled) .Values.podDisruptionBudget.enabled }}
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: {{ include "myapp.fullname" . }}
spec:
  {{- if .Values.podDisruptionBudget.minAvailable }}
  minAvailable: {{ .Values.podDisruptionBudget.minAvailable }}
  {{- end }}
  {{- if .Values.podDisruptionBudget.maxUnavailable }}
  maxUnavailable: {{ .Values.podDisruptionBudget.maxUnavailable }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "myapp.selectorLabels" . | nindent 6 }}
{{- end }}
```

### Demo-17: Helm Flow Control WITH Action

#### WITH 语句详解

WITH 语句用于改变作用域上下文，简化对嵌套对象的访问。

**WITH 语法：**
```yaml
{{- with .Values.nestedObject }}
# 在这里 . 代表 .Values.nestedObject
{{- end }}
```

**实际应用示例：**

**1. 简化嵌套对象访问：**
```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "myapp.fullname" . }}
spec:
  template:
    spec:
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
      
      containers:
      - name: {{ .Chart.Name }}
        {{- with .Values.resources }}
        resources:
          {{- toYaml . | nindent 10 }}
        {{- end }}
        
        {{- with .Values.securityContext }}
        securityContext:
          {{- toYaml . | nindent 10 }}
        {{- end }}
```

**2. 配置块处理：**
```yaml
# templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "myapp.fullname" . }}-config
data:
  {{- with .Values.database }}
  database.properties: |
    host={{ .host | default "localhost" }}
    port={{ .port | default 5432 }}
    name={{ .name | default $.Chart.Name }}
    ssl={{ .ssl | default false }}
    {{- with .pool }}
    pool.min={{ .min | default 5 }}
    pool.max={{ .max | default 20 }}
    pool.timeout={{ .timeout | default "30s" }}
    {{- end }}
  {{- end }}
  
  {{- with .Values.redis }}
  redis.properties: |
    host={{ .host | default "localhost" }}
    port={{ .port | default 6379 }}
    password={{ .password | default "" }}
    db={{ .db | default 0 }}
  {{- end }}
```

### Demo-18: Helm WITH Action and IfElse in Combination

#### WITH 和 IF-ELSE 组合使用

组合使用 WITH 和 IF-ELSE 可以创建更复杂和灵活的模板逻辑。

**实际应用示例：**

**1. 条件配置块：**
```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "myapp.fullname" . }}
spec:
  template:
    spec:
      containers:
      - name: {{ .Chart.Name }}
        env:
        {{- with .Values.database }}
        {{- if .enabled }}
        - name: DB_HOST
          value: {{ .host | quote }}
        - name: DB_PORT
          value: {{ .port | toString | quote }}
        - name: DB_NAME
          value: {{ .name | default $.Chart.Name | quote }}
        {{- if .ssl }}
        - name: DB_SSL_MODE
          value: "require"
        {{- else }}
        - name: DB_SSL_MODE
          value: "disable"
        {{- end }}
        {{- if .credentials }}
        {{- if .credentials.secret }}
        - name: DB_USERNAME
          valueFrom:
            secretKeyRef:
              name: {{ .credentials.secret.name }}
              key: {{ .credentials.secret.usernameKey | default "username" }}
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: {{ .credentials.secret.name }}
              key: {{ .credentials.secret.passwordKey | default "password" }}
        {{- else }}
        - name: DB_USERNAME
          value: {{ .credentials.username | quote }}
        - name: DB_PASSWORD
          value: {{ .credentials.password | quote }}
        {{- end }}
        {{- end }}
        {{- end }}
        {{- end }}
```

**2. 复杂服务配置：**
```yaml
# templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "myapp.fullname" . }}
  {{- with .Values.service }}
  {{- if .annotations }}
  annotations:
    {{- toYaml .annotations | nindent 4 }}
  {{- end }}
  {{- end }}
spec:
  {{- with .Values.service }}
  type: {{ .type | default "ClusterIP" }}
  {{- if eq .type "LoadBalancer" }}
  {{- if .loadBalancerIP }}
  loadBalancerIP: {{ .loadBalancerIP }}
  {{- end }}
  {{- if .loadBalancerSourceRanges }}
  loadBalancerSourceRanges:
  {{- range .loadBalancerSourceRanges }}
  - {{ . }}
  {{- end }}
  {{- end }}
  {{- else if eq .type "NodePort" }}
  {{- if .nodePort }}
  ports:
  - port: {{ .port | default 80 }}
    targetPort: {{ .targetPort | default "http" }}
    protocol: {{ .protocol | default "TCP" }}
    nodePort: {{ .nodePort }}
  {{- end }}
  {{- else }}
  ports:
  - port: {{ .port | default 80 }}
    targetPort: {{ .targetPort | default "http" }}
    protocol: {{ .protocol | default "TCP" }}
  {{- end }}
  {{- end }}
```

### Demo-19: Helm Variables

#### 变量定义和使用

Helm 模板支持变量定义，可以简化复杂的表达式和提高模板的可读性。

**变量语法：**
```yaml
{{- $variableName := .Values.someValue }}
{{- $result := include "myapp.fullname" . }}
```

**实际应用示例：**

**1. 复杂计算的变量：**
```yaml
# templates/deployment.yaml
{{- $fullName := include "myapp.fullname" . }}
{{- $labels := include "myapp.labels" . }}
{{- $selectorLabels := include "myapp.selectorLabels" . }}
{{- $serviceAccountName := include "myapp.serviceAccountName" . }}
{{- $imageName := printf "%s:%s" .Values.image.repository (.Values.image.tag | default .Chart.AppVersion) }}

apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ $fullName }}
  labels:
    {{- $labels | nindent 4 }}
spec:
  selector:
    matchLabels:
      {{- $selectorLabels | nindent 6 }}
  template:
    metadata:
      labels:
        {{- $selectorLabels | nindent 8 }}
    spec:
      serviceAccountName: {{ $serviceAccountName }}
      containers:
      - name: {{ .Chart.Name }}
        image: {{ $imageName }}
```

**2. 条件变量：**
```yaml
# templates/configmap.yaml
{{- $environment := .Values.environment | default "development" }}
{{- $logLevel := ternary "debug" "info" (eq $environment "development") }}
{{- $dbHost := ternary "localhost" .Values.database.host (eq $environment "development") }}

apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "myapp.fullname" . }}-config
data:
  environment: {{ $environment | quote }}
  log-level: {{ $logLevel | quote }}
  database-host: {{ $dbHost | quote }}
  
  app.properties: |
    app.environment={{ $environment }}
    app.log.level={{ $logLevel }}
    database.host={{ $dbHost }}
    database.port={{ .Values.database.port | default 5432 }}
```

**3. 循环中的变量：**
```yaml
# templates/configmap.yaml
{{- $root := . }}
{{- $fullName := include "myapp.fullname" . }}
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ $fullName }}-env-config
data:
{{- range $env, $config := .Values.environments }}
  {{- $envName := printf "%s-%s" $fullName $env }}
  {{ $env }}.properties: |
    app.name={{ $root.Chart.Name }}
    app.version={{ $root.Chart.AppVersion }}
    environment={{ $env }}
    {{- range $key, $value := $config }}
    {{ $key }}={{ $value }}
    {{- end }}
{{- end }}
```

### Demo-20: Helm Flow Control RANGE Action with List

#### RANGE 遍历列表

RANGE 用于遍历列表（数组），对每个元素执行模板逻辑。

**RANGE 语法：**
```yaml
{{- range .Values.list }}
# 这里 . 代表当前列表元素
{{- end }}

{{- range $index, $value := .Values.list }}
# $index 是索引，$value 是值
{{- end }}
```

**实际应用示例：**

**1. 多端口服务：**
```yaml
# values.yaml
service:
  ports:
  - name: http
    port: 80
    targetPort: 8080
    protocol: TCP
  - name: https
    port: 443
    targetPort: 8443
    protocol: TCP
  - name: metrics
    port: 9090
    targetPort: 9090
    protocol: TCP

# templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "myapp.fullname" . }}
spec:
  type: {{ .Values.service.type | default "ClusterIP" }}
  ports:
  {{- range .Values.service.ports }}
  - name: {{ .name }}
    port: {{ .port }}
    targetPort: {{ .targetPort }}
    protocol: {{ .protocol | default "TCP" }}
    {{- if and (eq $.Values.service.type "NodePort") .nodePort }}
    nodePort: {{ .nodePort }}
    {{- end }}
  {{- end }}
  selector:
    {{- include "myapp.selectorLabels" . | nindent 4 }}
```

**2. 多个 Ingress 主机：**
```yaml
# values.yaml
ingress:
  enabled: true
  hosts:
  - host: api.example.com
    paths:
    - path: /
      pathType: Prefix
    - path: /api
      pathType: Prefix
  - host: admin.example.com
    paths:
    - path: /
      pathType: Prefix

# templates/ingress.yaml
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "myapp.fullname" . }}
spec:
  rules:
  {{- range .Values.ingress.hosts }}
  - host: {{ .host | quote }}
    http:
      paths:
      {{- range .paths }}
      - path: {{ .path }}
        pathType: {{ .pathType }}
        backend:
          service:
            name: {{ include "myapp.fullname" $ }}
            port:
              number: {{ $.Values.service.port }}
      {{- end }}
  {{- end }}
{{- end }}
```

**3. 环境变量列表：**
```yaml
# values.yaml
env:
- name: APP_ENV
  value: production
- name: LOG_LEVEL
  value: info
- name: DATABASE_URL
  valueFrom:
    secretKeyRef:
      name: db-secret
      key: url

# templates/deployment.yaml
spec:
  template:
    spec:
      containers:
      - name: {{ .Chart.Name }}
        env:
        {{- range .Values.env }}
        - name: {{ .name }}
          {{- if .value }}
          value: {{ .value | quote }}
          {{- else if .valueFrom }}
          valueFrom:
            {{- toYaml .valueFrom | nindent 12 }}
          {{- end }}
        {{- end }}
```

### Demo-21: Helm Flow Control RANGE Action with Dictionary

#### RANGE 遍历字典

RANGE 也可以用于遍历字典（map），获取键值对。

**字典 RANGE 语法：**
```yaml
{{- range $key, $value := .Values.dictionary }}
# $key 是键，$value 是值
{{- end }}
```

**实际应用示例：**

**1. 动态配置文件：**
```yaml
# values.yaml
config:
  database:
    host: "localhost"
    port: 5432
    ssl: true
  redis:
    host: "redis-server"
    port: 6379
    db: 0
  app:
    workers: 4
    timeout: "30s"
    debug: false

# templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "myapp.fullname" . }}-config
data:
{{- range $section, $config := .Values.config }}
  {{ $section }}.properties: |
  {{- range $key, $value := $config }}
    {{ $key }}={{ $value }}
  {{- end }}
{{- end }}
```

**2. 多环境标签：**
```yaml
# values.yaml
labels:
  app: myapp
  version: v1.0.0
  environment: production
  team: backend
  
annotations:
  deployment.kubernetes.io/revision: "1"
  kubernetes.io/managed-by: helm

# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
    {{- range $key, $value := .Values.labels }}
    {{ $key }}: {{ $value | quote }}
    {{- end }}
  annotations:
    {{- range $key, $value := .Values.annotations }}
    {{ $key }}: {{ $value | quote }}
    {{- end }}
```

**3. 多个 Secret 挂载：**
```yaml
# values.yaml
secrets:
  database:
    name: db-credentials
    keys:
      username: db-username
      password: db-password
  api:
    name: api-keys
    keys:
      key: api-key
      secret: api-secret

# templates/deployment.yaml
spec:
  template:
    spec:
      containers:
      - name: {{ .Chart.Name }}
        env:
        {{- range $secretName, $secretConfig := .Values.secrets }}
        {{- range $envName, $keyName := $secretConfig.keys }}
        - name: {{ printf "%s_%s" ($secretName | upper) ($envName | upper) }}
          valueFrom:
            secretKeyRef:
              name: {{ $secretConfig.name }}
              key: {{ $keyName }}
        {{- end }}
        {{- end }}
```

**4. 复杂的存储卷配置：**
```yaml
# values.yaml
volumes:
  config:
    type: configMap
    name: app-config
    mountPath: /etc/config
  secrets:
    type: secret
    name: app-secrets
    mountPath: /etc/secrets
  data:
    type: persistentVolumeClaim
    name: app-data
    mountPath: /var/data

# templates/deployment.yaml
spec:
  template:
    spec:
      volumes:
      {{- range $volumeName, $volumeConfig := .Values.volumes }}
      - name: {{ $volumeName }}
        {{- if eq $volumeConfig.type "configMap" }}
        configMap:
          name: {{ $volumeConfig.name }}
        {{- else if eq $volumeConfig.type "secret" }}
        secret:
          secretName: {{ $volumeConfig.name }}
        {{- else if eq $volumeConfig.type "persistentVolumeClaim" }}
        persistentVolumeClaim:
          claimName: {{ $volumeConfig.name }}
        {{- end }}
      {{- end }}
      
      containers:
      - name: {{ .Chart.Name }}
        volumeMounts:
        {{- range $volumeName, $volumeConfig := .Values.volumes }}
        - name: {{ $volumeName }}
          mountPath: {{ $volumeConfig.mountPath }}
          {{- if $volumeConfig.subPath }}
          subPath: {{ $volumeConfig.subPath }}
          {{- end }}
          {{- if $volumeConfig.readOnly }}
          readOnly: {{ $volumeConfig.readOnly }}
          {{- end }}
        {{- end }}
```

### 高级控制结构综合示例

结合所有高级控制结构的完整示例：

```yaml
# templates/deployment.yaml
{{- $fullName := include "myapp.fullname" . }}
{{- $labels := include "myapp.labels" . }}
{{- $environment := .Values.environment | default "development" }}
{{- $isProduction := eq $environment "production" }}
{{- $hasDatabase := and .Values.database .Values.database.enabled }}

apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ $fullName }}
  labels:
    {{- $labels | nindent 4 }}
    environment: {{ $environment | quote }}
spec:
  {{- if and $isProduction (not .Values.autoscaling.enabled) }}
  replicas: {{ .Values.replicaCount | default 3 }}
  {{- else if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount | default 1 }}
  {{- end }}
  
  selector:
    matchLabels:
      {{- include "myapp.selectorLabels" . | nindent 6 }}
  
  template:
    metadata:
      labels:
        {{- include "myapp.selectorLabels" . | nindent 8 }}
        environment: {{ $environment | quote }}
      annotations:
        {{- if .Values.podAnnotations }}
        {{- range $key, $value := .Values.podAnnotations }}
        {{ $key }}: {{ $value | quote }}
        {{- end }}
        {{- end }}
    
    spec:
      {{- with .Values.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
        imagePullPolicy: {{ .Values.image.pullPolicy | default "IfNotPresent" }}
        
        ports:
        {{- range .Values.service.ports }}
        - name: {{ .name }}
          containerPort: {{ .targetPort | default .port }}
          protocol: {{ .protocol | default "TCP" }}
        {{- end }}
        
        env:
        - name: ENVIRONMENT
          value: {{ $environment | quote }}
        {{- if $hasDatabase }}
        {{- with .Values.database }}
        - name: DB_HOST
          value: {{ .host | quote }}
        - name: DB_PORT
          value: {{ .port | toString | quote }}
        - name: DB_NAME
          value: {{ .name | default $.Chart.Name | quote }}
        {{- if .credentials }}
        {{- if .credentials.secret }}
        - name: DB_USERNAME
          valueFrom:
            secretKeyRef:
              name: {{ .credentials.secret.name }}
              key: {{ .credentials.secret.usernameKey | default "username" }}
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: {{ .credentials.secret.name }}
              key: {{ .credentials.secret.passwordKey | default "password" }}
        {{- end }}
        {{- end }}
        {{- end }}
        {{- end }}
        {{- if .Values.extraEnv }}
        {{- range $key, $value := .Values.extraEnv }}
        - name: {{ $key | upper }}
          value: {{ $value | quote }}
        {{- end }}
        {{- end }}
        
        {{- if or $isProduction .Values.probes.enabled }}
        livenessProbe:
          httpGet:
            path: {{ .Values.livenessProbe.path | default "/health" }}
            port: http
          initialDelaySeconds: {{ .Values.livenessProbe.initialDelaySeconds | default 30 }}
          periodSeconds: {{ .Values.livenessProbe.periodSeconds | default 10 }}
        
        readinessProbe:
          httpGet:
            path: {{ .Values.readinessProbe.path | default "/ready" }}
            port: http
          initialDelaySeconds: {{ .Values.readinessProbe.initialDelaySeconds | default 5 }}
          periodSeconds: {{ .Values.readinessProbe.periodSeconds | default 5 }}
        {{- end }}
        
        {{- if or $isProduction .Values.resources }}
        resources:
          {{- if $isProduction }}
          limits:
            cpu: {{ .Values.resources.limits.cpu | default "1000m" }}
            memory: {{ .Values.resources.limits.memory | default "1Gi" }}
          requests:
            cpu: {{ .Values.resources.requests.cpu | default "500m" }}
            memory: {{ .Values.resources.requests.memory | default "512Mi" }}
          {{- else }}
          {{- toYaml .Values.resources | nindent 10 }}
          {{- end }}
        {{- end }}
        
        {{- if .Values.volumeMounts }}
        volumeMounts:
        {{- range $mountName, $mountConfig := .Values.volumeMounts }}
        - name: {{ $mountName }}
          mountPath: {{ $mountConfig.mountPath }}
          {{- if $mountConfig.subPath }}
          subPath: {{ $mountConfig.subPath }}
          {{- end }}
          {{- if $mountConfig.readOnly }}
          readOnly: {{ $mountConfig.readOnly }}
          {{- end }}
        {{- end }}
        {{- end }}
      
      {{- if .Values.volumes }}
      volumes:
      {{- range $volumeName, $volumeConfig := .Values.volumes }}
      - name: {{ $volumeName }}
        {{- if eq $volumeConfig.type "configMap" }}
        configMap:
          name: {{ $volumeConfig.name }}
        {{- else if eq $volumeConfig.type "secret" }}
        secret:
          secretName: {{ $volumeConfig.name }}
        {{- else if eq $volumeConfig.type "persistentVolumeClaim" }}
        persistentVolumeClaim:
          claimName: {{ $volumeConfig.name }}
        {{- else if eq $volumeConfig.type "emptyDir" }}
        emptyDir: {}
        {{- end }}
      {{- end }}
      {{- end }}
      
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

---

## Override Values (值覆盖)

### 什么是 Override Values？

Override Values 是 Helm 中用于自定义 Chart 默认配置的机制。它允许您在不修改原始 Chart 的情况下，覆盖 `values.yaml` 中的默认值，实现个性化部署。

### 为什么需要 Override Values？

1. **环境差异化**：不同环境（开发、测试、生产）需要不同的配置
2. **复用性**：同一个 Chart 可以部署多个实例，每个实例有不同配置
3. **安全性**：避免直接修改 Chart 源码，保持更新能力
4. **灵活性**：快速调整配置而不需要重新打包 Chart

### 如何使用 Override Values？

#### 1. 基础语法

```bash
# 使用 --set 覆盖单个值
helm install myapp ./mychart --set image.tag=v2.0

# 使用 --values 或 -f 指定自定义 values 文件
helm install myapp ./mychart -f custom-values.yaml

# 组合使用多个 values 文件
helm install myapp ./mychart -f base-values.yaml -f env-specific.yaml
```

#### 2. --set 参数详解

```bash
# 简单值覆盖
helm install app ./chart --set replicaCount=3

# 嵌套值覆盖
helm install app ./chart --set image.repository=nginx --set image.tag=1.20

# 数组值覆盖
helm install app ./chart --set ingress.hosts[0].host=example.com

# 布尔值覆盖
helm install app ./chart --set service.enabled=true

# 复杂对象覆盖
helm install app ./chart \
  --set service.type=LoadBalancer \
  --set service.port=8080 \
  --set resources.limits.cpu=500m \
  --set resources.limits.memory=512Mi
```

#### 3. Values 文件覆盖

创建自定义 values 文件：

```yaml
# custom-values.yaml
replicaCount: 3

image:
  repository: nginx
  tag: "1.20"
  pullPolicy: Always

service:
  type: LoadBalancer
  port: 80

resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 250m
    memory: 256Mi

ingress:
  enabled: true
  hosts:
    - host: myapp.example.com
      paths:
        - path: /
          pathType: Prefix
```

```bash
helm install myapp ./mychart -f custom-values.yaml
```

#### 4. 优先级顺序

Helm 按以下优先级处理值（后者覆盖前者）：

1. Chart 的默认 `values.yaml`
2. 父 Chart 的 `values.yaml`（如果是子 Chart）
3. 用户提供的 values 文件（`-f` 参数）
4. `--set` 参数

```bash
# 演示优先级
helm install app ./chart \
  -f base-values.yaml \      # 优先级 3
  -f override-values.yaml \  # 优先级 3（后面的覆盖前面的）
  --set image.tag=latest     # 优先级 4（最高）
```

---

## --dry-run (模拟运行)

### 什么是 --dry-run？

`--dry-run` 是 Helm 的模拟运行模式，它会执行所有的模板渲染和验证过程，但不会实际向 Kubernetes 集群提交任何资源。

### 为什么使用 --dry-run？

1. **预览部署**：在实际部署前查看将要创建的资源
2. **验证配置**：确认 values 覆盖是否正确
3. **调试模板**：检查模板渲染结果
4. **安全性**：避免错误部署影响生产环境
5. **学习工具**：理解 Chart 的工作原理

### 如何使用 --dry-run？

#### 1. 基础用法

```bash
# 模拟安装
helm install myapp ./mychart --dry-run

# 模拟升级
helm upgrade myapp ./mychart --dry-run

# 结合 values 覆盖
helm install myapp ./mychart -f custom-values.yaml --dry-run
```

#### 2. 输出解析

```bash
helm install webapp ./web-chart --dry-run
```

输出示例：
```yaml
NAME: webapp
LAST DEPLOYED: Tue Jan  1 00:00:00 2024
NAMESPACE: default
STATUS: pending-install
REVISION: 1
TEST SUITE: None
HOOKS:
---
# Source: web-chart/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  labels:
    app: webapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: webapp
        image: nginx:1.19
        ports:
        - containerPort: 80
---
# Source: web-chart/templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: webapp
spec:
  type: ClusterIP
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: webapp
```

#### 3. 高级用法

```bash
# 只渲染特定模板
helm template myapp ./mychart --show-only templates/deployment.yaml

# 验证但不输出渲染结果
helm install myapp ./mychart --dry-run > /dev/null && echo "验证通过"

# 结合其他参数
helm install myapp ./mychart \
  --dry-run \
  --debug \
  --set replicaCount=3 \
  -f production-values.yaml
```

---

## --debug (调试模式)

### 什么是 --debug？

`--debug` 启用 Helm 的调试模式，提供详细的执行信息，包括模板渲染过程、变量值、错误详情等。

### 为什么使用 --debug？

1. **故障排查**：定位部署失败的原因
2. **模板调试**：查看变量如何传递和处理
3. **性能分析**：了解 Helm 的执行过程
4. **学习目的**：深入理解 Helm 工作机制

### 如何使用 --debug？

#### 1. 基础调试

```bash
# 基础调试信息
helm install myapp ./mychart --debug

# 结合 dry-run 进行安全调试
helm install myapp ./mychart --dry-run --debug
```

#### 2. 调试输出解析

```bash
helm install webapp ./web-chart --dry-run --debug
```

输出示例：
```
install.go:178: [debug] Original chart version: ""
install.go:195: [debug] CHART PATH: ./web-chart

NAME: webapp
LAST DEPLOYED: Tue Jan  1 00:00:00 2024
NAMESPACE: default
STATUS: pending-install
REVISION: 1
USER-SUPPLIED VALUES:
{}

COMPUTED VALUES:
affinity: {}
autoscaling:
  enabled: false
  maxReplicas: 100
  minReplicas: 1
  targetCPUUtilizationPercentage: 80
fullnameOverride: ""
image:
  pullPolicy: IfNotPresent
  repository: nginx
  tag: "1.19"
imagePullSecrets: []
ingress:
  annotations: {}
  className: ""
  enabled: false
  hosts:
  - host: chart-example.local
    paths:
    - path: /
      pathType: ImplementationSpecific
  tls: []
nameOverride: ""
nodeSelector: {}
podAnnotations: {}
podSecurityContext: {}
replicaCount: 1
resources: {}
securityContext: {}
service:
  port: 80
  type: ClusterIP
serviceAccount:
  annotations: {}
  create: true
  name: ""
tolerations: []

HOOKS:
MANIFEST:
---
# Source: web-chart/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
# ... 完整的渲染结果
```

#### 3. 调试特定问题

```bash
# 调试模板语法错误
helm install myapp ./mychart --debug --dry-run 2>&1 | grep -i error

# 调试 values 传递问题
helm install myapp ./mychart --debug --set debug=true --dry-run

# 调试依赖问题
helm dependency update ./mychart --debug
```

---

## helm get (获取资源信息)

### 什么是 helm get？

`helm get` 命令用于检索已部署 release 的信息，包括配置值、manifest、notes 等。

### 为什么使用 helm get？

1. **状态检查**：了解当前部署的状态
2. **配置审计**：查看实际使用的配置值
3. **故障排查**：分析部署问题
4. **文档目的**：生成部署文档

### 如何使用 helm get？

#### 1. 子命令概览

```bash
# 获取所有信息
helm get all RELEASE_NAME

# 获取 computed values
helm get values RELEASE_NAME

# 获取 manifest
helm get manifest RELEASE_NAME

# 获取 notes
helm get notes RELEASE_NAME

# 获取 hooks
helm get hooks RELEASE_NAME
```

#### 2. helm get values

```bash
# 获取当前使用的所有 values
helm get values webapp

# 获取用户提供的 values（排除默认值）
helm get values webapp --user-defined

# 指定输出格式
helm get values webapp -o yaml
helm get values webapp -o json

# 获取特定版本的 values
helm get values webapp --revision 2
```

输出示例：
```yaml
USER-SUPPLIED VALUES:
image:
  tag: v2.0
replicaCount: 3
service:
  type: LoadBalancer
```

#### 3. helm get manifest

```bash
# 获取完整的 Kubernetes manifest
helm get manifest webapp

# 保存到文件
helm get manifest webapp > webapp-manifest.yaml

# 获取特定版本的 manifest
helm get manifest webapp --revision 1
```

#### 4. helm get notes

```bash
# 获取安装后的提示信息
helm get notes webapp
```

输出示例：
```
NOTES:
1. Get the application URL by running these commands:
   export SERVICE_IP=$(kubectl get svc --namespace default webapp --template "{{ range (index .status.loadBalancer.ingress 0) }}{{.}}{{ end }}")
   echo http://$SERVICE_IP:80

2. To access the application:
   kubectl port-forward svc/webapp 8080:80
   Then visit http://localhost:8080
```

#### 5. helm get hooks

```bash
# 获取所有 hooks
helm get hooks webapp

# 指定输出格式
helm get hooks webapp -o yaml
```

#### 6. helm get all

```bash
# 获取 release 的完整信息
helm get all webapp

# 指定版本
helm get all webapp --revision 2

# 输出到文件
helm get all webapp > webapp-complete-info.txt
```

---

## 综合实践案例

### 案例 1：多环境部署

#### 1. 准备环境特定的 values 文件

```yaml
# values-dev.yaml
replicaCount: 1
image:
  tag: "dev-latest"
service:
  type: ClusterIP
resources:
  limits:
    cpu: 200m
    memory: 256Mi

ingress:
  enabled: true
  hosts:
    - host: myapp-dev.internal
```

```yaml
# values-prod.yaml
replicaCount: 3
image:
  tag: "v1.2.3"
service:
  type: LoadBalancer
resources:
  limits:
    cpu: 1000m
    memory: 1Gi

ingress:
  enabled: true
  hosts:
    - host: myapp.company.com
```

#### 2. 部署流程

```bash
# 开发环境部署（先预览）
helm install myapp-dev ./mychart \
  -f values-dev.yaml \
  --namespace dev \
  --dry-run --debug

# 确认无误后实际部署
helm install myapp-dev ./mychart \
  -f values-dev.yaml \
  --namespace dev

# 生产环境部署
helm install myapp-prod ./mychart \
  -f values-prod.yaml \
  --namespace production \
  --dry-run

helm install myapp-prod ./mychart \
  -f values-prod.yaml \
  --namespace production
```

#### 3. 部署后验证

```bash
# 检查部署状态
helm get values myapp-dev --namespace dev
helm get values myapp-prod --namespace production

# 对比配置差异
helm get values myapp-dev --namespace dev > dev-values.yaml
helm get values myapp-prod --namespace production > prod-values.yaml
diff dev-values.yaml prod-values.yaml
```

### 案例 2：复杂配置调试

#### 1. 问题场景

假设您在部署一个复杂的应用时遇到错误：

```bash
helm install complex-app ./complex-chart -f production.yaml
```

错误输出：
```
Error: unable to build kubernetes objects from release manifest: error validating "": error validating data: ValidationError(Deployment.spec.template.spec.containers[0].env[3].valueFrom.secretKeyRef): missing required field "name"
```

#### 2. 调试过程

```bash
# 第一步：使用 dry-run 和 debug 查看完整输出
helm install complex-app ./complex-chart \
  -f production.yaml \
  --dry-run --debug > debug-output.yaml

# 第二步：检查问题区域
grep -A 10 -B 10 "secretKeyRef" debug-output.yaml

# 第三步：检查 values 传递
helm get values complex-app --dry-run -f production.yaml

# 第四步：修复配置后重新验证
helm install complex-app ./complex-chart \
  -f production.yaml \
  --set secrets.database.name=db-secret \
  --dry-run --debug
```

### 案例 3：升级配置管理

#### 1. 升级前检查

```bash
# 查看当前配置
helm get values myapp

# 查看当前 manifest
helm get manifest myapp > current-manifest.yaml

# 模拟升级
helm upgrade myapp ./mychart \
  -f new-values.yaml \
  --dry-run --debug > upgrade-preview.yaml

# 对比差异
diff current-manifest.yaml upgrade-preview.yaml
```

#### 2. 安全升级

```bash
# 实际升级
helm upgrade myapp ./mychart -f new-values.yaml

# 升级后验证
helm get values myapp --revision 2
helm get notes myapp
```

---

## 最佳实践

### 1. Override Values 最佳实践

```bash
# ✅ 好的做法
# 使用环境特定的 values 文件
helm install app ./chart -f values-production.yaml

# 将敏感信息通过 --set 传递（从环境变量）
helm install app ./chart --set database.password=$DB_PASSWORD

# 使用层次化的 values 文件
helm install app ./chart \
  -f values-base.yaml \
  -f values-environment.yaml \
  -f values-secrets.yaml

# ❌ 避免的做法
# 不要在命令行中暴露敏感信息
helm install app ./chart --set database.password=plaintext123

# 不要使用过于复杂的 --set 语法
helm install app ./chart --set 'ingress.hosts[0].host=example.com,ingress.hosts[0].paths[0].path=/'
```

### 2. 调试最佳实践

```bash
# ✅ 渐进式调试
# 1. 先使用 dry-run 验证
helm install app ./chart --dry-run

# 2. 遇到问题时添加 debug
helm install app ./chart --dry-run --debug

# 3. 检查特定模板
helm template app ./chart --show-only templates/deployment.yaml

# 4. 验证 values 传递
helm template app ./chart --debug | grep -A 20 "COMPUTED VALUES"

# ✅ 错误处理
# 将调试信息保存到文件
helm install app ./chart --dry-run --debug 2>&1 | tee debug.log

# 分析错误
grep -i error debug.log
grep -i warning debug.log
```

### 3. 生产环境最佳实践

```bash
# ✅ 生产部署流程
# 1. 本地验证
helm template myapp ./mychart -f production-values.yaml > production-manifest.yaml
kubectl apply --dry-run=client -f production-manifest.yaml

# 2. 集群验证
helm install myapp ./mychart -f production-values.yaml --dry-run

# 3. 实际部署
helm install myapp ./mychart -f production-values.yaml

# 4. 部署后验证
helm get all myapp
kubectl get pods -l app=myapp
```

### 4. 监控和维护

```bash
# 定期检查部署状态
helm list --all-namespaces

# 检查特定应用的历史
helm history myapp

# 获取当前配置（用于备份）
helm get values myapp > myapp-current-values.yaml
helm get manifest myapp > myapp-current-manifest.yaml

# 清理测试部署
helm uninstall test-deployment --dry-run
```

### 5. 团队协作最佳实践

```yaml
# 在项目中维护标准的 values 文件结构
values/
├── values-base.yaml          # 基础配置
├── values-development.yaml   # 开发环境
├── values-staging.yaml       # 测试环境
├── values-production.yaml    # 生产环境
└── values-secrets.yaml.example  # 敏感信息模板
```

```bash
# 使用 Makefile 标准化部署流程
# Makefile
deploy-dev:
	helm upgrade --install myapp ./chart -f values/values-base.yaml -f values/values-development.yaml --namespace dev

deploy-prod:
	helm upgrade --install myapp ./chart -f values/values-base.yaml -f values/values-production.yaml --namespace production --dry-run
	@echo "Review the above output, then run: make deploy-prod-confirm"

deploy-prod-confirm:
	helm upgrade --install myapp ./chart -f values/values-base.yaml -f values/values-production.yaml --namespace production
```

---

## 总结

掌握 Helm 的 Override Values、--dry-run、--debug 和 get 命令是成功使用 Helm 的关键：

1. **Override Values** 提供了灵活的配置管理机制
2. **--dry-run** 确保部署安全性和准确性
3. **--debug** 帮助快速定位和解决问题
4. **helm get** 提供了全面的部署状态查询能力

结合这些工具，您可以构建稳定、可维护的 Kubernetes 应用部署流程。
