# Helm 完整学习指南

本文档整合了 Helm 的完整知识体系，包括实战故事、核心优势、基础操作和高级用法，旨在帮助读者全面掌握 Helm 的使用。

## 📚 目录

1. [从故事开始：为什么需要 Helm](#1-从故事开始为什么需要-helm)
2. [Helm 的核心优势](#2-helm-的核心优势)
3. [Helm 基础入门](#3-helm-基础入门)
4. [实战操作指南](#4-实战操作指南)
5. [为自己的项目创建 Helm Chart](#5-为自己的项目创建-helm-chart)
6. [高级特性和最佳实践](#6-高级特性和最佳实践)

---

## 1. 从故事开始：为什么需要 Helm

### 1.1 风暴前夕

深夜 11 点，DevOps 工程师小李盯着满屏的 YAML 文件，眼睛已经开始发酸。这是他连续第三个通宵加班了。

"又是 ConfigMap 配置错误！" 小李沮丧地敲击着键盘。他们的电商平台需要紧急部署一个促销活动的新功能，但是在将应用从测试环境部署到生产环境时，问题接踵而至。

### 1.2 手动部署的噩梦

#### 配置地狱

团队需要管理的 Kubernetes 资源清单包括：
- 15 个微服务的 Deployment 配置
- 各种 Service 和 Ingress 规则
- 数十个 ConfigMap 和 Secret
- PersistentVolumeClaim 存储配置
- HorizontalPodAutoscaler 自动扩缩容规则

每次部署都需要手动修改这些 YAML 文件：

```yaml
# Order service deployment - production version
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: production  # 需要手动从 'staging' 改为 'production'
spec:
  replicas: 3  # 需要手动从测试的 1 个改为生产的 3 个
  template:
    spec:
      containers:
      - name: order-service
        image: mycompany/order-service:v2.3.1  # 需要手动更新版本号
        env:
        - name: DATABASE_URL
          value: "postgresql://prod-db.company.com:5432/orders"  # 需要手动修改数据库地址
        - name: REDIS_HOST
          value: "prod-redis.company.com"  # 需要手动修改 Redis 地址
```

#### 版本混乱

"等等，支付服务应该用 v2.1.0 还是 v2.1.1？" 前端工程师小王在群里问道。

"我记得昨天测试的是 v2.1.0，但是今天早上修复了一个 bug..." 后端开发小张回复。

没有统一的版本管理，团队成员经常搞不清楚哪个版本在哪个环境运行。更糟糕的是，有时候部署了错误的版本组合，导致服务之间的 API 不兼容。

#### 回滚灾难

凌晨 2 点，监控系统突然报警 - 订单服务响应时间飙升到 5 秒以上！

"快回滚！" 技术经理在电话里焦急地说。

但是回滚谈何容易？团队需要：
1. 找出上一个稳定版本的所有配置文件
2. 手动将每个服务的镜像标签改回旧版本
3. 重新应用所有的 YAML 文件
4. 祈祷没有遗漏任何配置

整个回滚过程耗时 45 分钟，期间损失了大量订单。

### 1.3 Helm - 经验丰富的舵手登场

就在团队濒临崩溃的时候，新来的 SRE 工程师小陈提出了一个建议："为什么不试试 Helm？"

"Helm？那是什么？" 大家好奇地问。

小陈打开了笔记本，展示了一个简洁的 Chart 结构：

```
my-app-chart/
├── Chart.yaml          # Chart 的基本信息
├── values.yaml         # 默认配置值
├── values-prod.yaml    # 生产环境配置
├── values-staging.yaml # 测试环境配置
└── templates/          # 模板文件
    ├── deployment.yaml
    ├── service.yaml
    ├── configmap.yaml
    └── ingress.yaml
```

"Helm 就像一个经验丰富的舵手，" 小陈解释道，"它能帮我们在 Kubernetes 的汪洋大海中稳定航行。"

#### 统一的版本管理

```yaml
# values-prod.yaml
appVersion: "2.3.1"
services:
  order:
    image:
      tag: "{{ .Values.appVersion }}"
    replicas: 3
  payment:
    image:
      tag: "{{ .Values.appVersion }}"
    replicas: 3
```

"只需要改一个地方，所有服务的版本就都更新了！" 小李眼睛一亮。

#### 环境差异的优雅处理

```bash
# Deploy to staging
helm upgrade --install my-app ./my-app-chart -f values-staging.yaml

# Deploy to production
helm upgrade --install my-app ./my-app-chart -f values-prod.yaml
```

"不同环境的配置完全分离，再也不用担心把测试配置部署到生产了！" 小王兴奋地说。

#### 一键回滚

当问题再次出现时：

```bash
# View deployment history
helm history my-app

# Rollback to previous version
helm rollback my-app 1

# Rollback completed in 30 seconds!
```

"30 秒完成回滚？这简直是魔法！" 技术经理难以置信。

### 1.4 风平浪静

三个月后，团队的部署流程已经完全改变：

- **部署时间**：从平均 2 小时缩短到 10 分钟
- **部署错误**：从每周 3-4 次降低到几乎为零
- **回滚时间**：从 45 分钟缩短到 30 秒
- **加班时间**：小李终于可以正常下班了

更重要的是，团队重新找回了信心。他们不再害怕部署，不再担心配置错误，因为他们知道，有 Helm 这个可靠的舵手在掌控方向。

---

## 2. Helm 的核心优势

Helm 是 Kubernetes 的包管理器，它能够大大简化 Kubernetes 应用的部署和管理。以下是使用 Helm 的主要优势：

### 2.1 简化部署流程

#### 传统方式的痛点
- 需要手动管理多个 YAML 文件
- 部署顺序需要人工控制
- 容易出现配置错误

#### Helm 的解决方案
- **一键部署**：通过一个简单的命令即可部署整个应用栈
  ```bash
  helm install my-app ./my-chart
  ```
- **预定义模板**：使用 Go 模板语言，减少重复配置
- **自动处理依赖关系**：按正确顺序部署资源

#### 实际案例
```yaml
# values.yaml
replicaCount: 3
image:
  repository: nginx
  tag: "1.16.0"
  pullPolicy: IfNotPresent

# deployment.yaml template
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-deployment
spec:
  replicas: {{ .Values.replicaCount }}
  template:
    spec:
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```

### 2.2 版本管理和回滚

#### 版本控制能力
- **Release 历史记录**：每次部署都会保存为一个 release
- **查看部署历史**：
  ```bash
  helm history my-release
  ```
- **对比版本差异**：可以查看不同版本之间的配置变化

#### 快速回滚机制
- **一键回滚**：出现问题时可以立即恢复到之前的版本
  ```bash
  helm rollback my-release 1
  ```
- **保留配置历史**：所有历史配置都被保存，可随时恢复
- **原子操作**：回滚是原子性的，要么全部成功，要么全部失败

### 2.3 配置的参数化和复用

#### 参数化配置
- **Values 文件**：将可变配置抽取到 values.yaml
- **环境差异管理**：为不同环境使用不同的 values 文件
  ```bash
  # Development environment
  helm install my-app ./my-chart -f values-dev.yaml
  
  # Production environment
  helm install my-app ./my-chart -f values-prod.yaml
  ```

#### 配置复用
- **模板继承**：使用 `_helpers.tpl` 定义可复用的模板片段
- **条件渲染**：根据参数动态生成配置
  ```yaml
  {{- if .Values.ingress.enabled -}}
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: {{ .Release.Name }}-ingress
  # ... ingress configuration
  {{- end }}
  ```

#### 配置覆盖
- **多层配置**：支持默认值、Chart 值、用户值的层级覆盖
- **命令行覆盖**：临时修改配置而不改变文件
  ```bash
  helm install my-app ./my-chart --set image.tag=1.17.0
  ```

### 2.4 依赖管理

#### Chart 依赖
- **声明式依赖**：在 Chart.yaml 中声明依赖
  ```yaml
  dependencies:
    - name: postgresql
      version: "11.6.12"
      repository: "https://charts.bitnami.com/bitnami"
    - name: redis
      version: "17.3.14"
      repository: "https://charts.bitnami.com/bitnami"
  ```

#### 依赖管理命令
- **更新依赖**：
  ```bash
  helm dependency update
  ```
- **下载依赖**：
  ```bash
  helm dependency build
  ```

#### 条件依赖
- **按需启用**：根据配置决定是否部署某个依赖
  ```yaml
  dependencies:
    - name: postgresql
      condition: postgresql.enabled
  ```

### 2.5 共享和分发应用

#### Chart 仓库
- **公共仓库**：使用 Artifact Hub 等公共仓库
- **私有仓库**：企业可以搭建私有 Chart 仓库
- **仓库管理**：
  ```bash
  # Add repository
  helm repo add bitnami https://charts.bitnami.com/bitnami
  
  # Search charts
  helm search repo wordpress
  
  # Update repository
  helm repo update
  ```

#### Chart 打包和分发
- **打包 Chart**：
  ```bash
  helm package ./my-chart
  ```
- **推送到仓库**：支持推送到 OCI 注册表
  ```bash
  helm push my-chart-0.1.0.tgz oci://registry.example.com/helm-charts
  ```

#### 版本化发布
- **语义化版本**：遵循 SemVer 规范
- **Chart 版本**：独立于应用版本的 Chart 版本管理
- **依赖锁定**：Chart.lock 文件确保依赖版本一致性

---

## 3. Helm 基础入门

### 3.1 安装 Helm

```bash
# macOS
brew install helm

# Linux
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# 验证安装
helm version
```

### 3.2 基本命令

#### 仓库管理

```bash
# Add a repository
helm repo add bitnami https://charts.bitnami.com/bitnami

# List repositories
helm repo list

# Update repositories
helm repo update

# Search for charts
helm search repo nginx
```

#### Chart 操作

```bash
# Create a new chart
helm create my-app

# Install a chart
helm install my-release bitnami/nginx

# List installed releases
helm list

# Get release status
helm status my-release
```

#### 升级和回滚

```bash
# Upgrade a release
helm upgrade my-release bitnami/nginx --set replicaCount=3

# View release history
helm history my-release

# Rollback to a previous revision
helm rollback my-release 1

# Uninstall a release
helm uninstall my-release
```

### 3.3 查看 Helm Release 的配置

当你需要查看已部署的 Helm Release 的配置时，可以使用以下命令：

#### 查看当前 Values

```bash
# View current values for a release
helm get values <release-name> -n <namespace>

# Example: View kube-prometheus-stack values
helm get values monitoring -n monitoring

# Save values to a file
helm get values monitoring -n monitoring > monitoring-values.yaml

# View all values (including defaults)
helm get values monitoring -n monitoring --all
```

#### 查看完整的 Release 信息

```bash
# Get all information about a release
helm get all <release-name> -n <namespace>

# Get manifest (rendered YAML)
helm get manifest monitoring -n monitoring

# Get notes
helm get notes monitoring -n monitoring

# Get hooks
helm get hooks monitoring -n monitoring
```

#### 比较配置差异

```bash
# Compare with default values
helm get values monitoring -n monitoring > current-values.yaml
helm show values prometheus-community/kube-prometheus-stack > default-values.yaml
diff default-values.yaml current-values.yaml
```

### 3.4 创建自定义 Chart

#### Chart 结构

```
my-app/
├── Chart.yaml          # Chart metadata
├── values.yaml         # Default configuration values
├── charts/            # Chart dependencies
├── templates/         # Template files
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── configmap.yaml
│   ├── secret.yaml
│   ├── NOTES.txt
│   └── _helpers.tpl
└── .helmignore        # Patterns to ignore
```

#### 示例 values.yaml

```yaml
# Default values for my-app
replicaCount: 1

image:
  repository: nginx
  pullPolicy: IfNotPresent
  tag: "1.21.0"

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: false
  className: "nginx"
  hosts:
    - host: chart-example.local
      paths:
        - path: /
          pathType: ImplementationSpecific

resources:
  limits:
    cpu: 100m
    memory: 128Mi
  requests:
    cpu: 100m
    memory: 128Mi

autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 100
  targetCPUUtilizationPercentage: 80
```

---

## 4. 实战操作指南

### 4.1 Kube-Prometheus-Stack 实战案例

本节以 `prometheus-community/kube-prometheus-stack` Helm Chart 为例，演示实际的操作流程。

#### Chart 信息说明

`prometheus-community/kube-prometheus-stack` 格式说明：
- **prometheus-community**: Helm 仓库名称
- **kube-prometheus-stack**: Chart 名称

#### 前置准备

##### 1. 添加 Helm 仓库

```bash
# Add the prometheus-community repository
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

# Update the repository list
helm repo update
```

##### 2. 查看已添加的仓库

```bash
# List all configured repositories
helm repo list
```

### 4.2 本地 Chart 安装

除了从远程仓库拉取 Helm Chart 进行安装，还可以使用本地的 Chart 目录进行安装。这种方式在以下场景中特别有用：
- 开发和测试自定义 Chart
- 在没有网络连接的环境中部署
- 需要对官方 Chart 进行修改后使用
- 管理私有的 Chart 集合

#### 基本用法

对于一个位于本地目录（如 `./mysql/`）中的 Chart，可以使用以下命令进行安装：

```bash
# Basic local chart installation
helm install mysql-local ./mysql/

# Install with namespace
helm install mysql-local ./mysql/ --namespace database --create-namespace

# Install with custom values
helm install mysql-local ./mysql/ --values my-values.yaml
```

#### 本地 Chart 目录结构

一个标准的 Helm Chart 目录结构如下：

```
mysql/
├── Chart.yaml          # Chart 的元数据信息
├── values.yaml         # 默认配置值
├── templates/          # Kubernetes 资源模板
│   ├── deployment.yaml
│   ├── service.yaml
│   └── ...
├── charts/            # 依赖的子 Chart（可选）
└── README.md          # Chart 说明文档（可选）
```

#### 本地 Chart 的常用操作

##### 1. 验证本地 Chart

在安装前，建议先验证 Chart 的正确性：

```bash
# Lint the local chart
helm lint ./mysql/

# Dry run to test the chart
helm install mysql-local ./mysql/ --dry-run --debug
```

##### 2. 查看本地 Chart 信息

```bash
# Show chart metadata
helm show chart ./mysql/

# Show default values
helm show values ./mysql/

# Show README
helm show readme ./mysql/
```

##### 3. 打包本地 Chart

如果需要分发或存档，可以将 Chart 打包：

```bash
# Package the chart
helm package ./mysql/

# Package with specific version
helm package ./mysql/ --version 1.2.3

# Package to specific directory
helm package ./mysql/ --destination ./packages/
```

##### 4. 从打包文件安装

```bash
# Install from packaged chart
helm install mysql-local mysql-1.2.3.tgz

# Install from URL
helm install mysql-local https://example.com/charts/mysql-1.2.3.tgz
```

#### 本地开发工作流示例

```bash
# 1. Create a new chart
helm create mychart

# 2. Modify the chart files
cd mychart
# Edit templates/, values.yaml, etc.

# 3. Test the chart
helm lint .
helm template . --debug

# 4. Install for testing
helm install test-release . --namespace test --create-namespace

# 5. Upgrade after changes
helm upgrade test-release . --namespace test

# 6. Clean up
helm uninstall test-release --namespace test
```

#### 本地 Chart 与远程仓库的结合使用

```bash
# Download a chart from repository without installing
helm pull prometheus-community/kube-prometheus-stack

# Extract the downloaded chart
tar -xzf kube-prometheus-stack-*.tgz

# Modify and install from local directory
cd kube-prometheus-stack/
# Make your modifications
helm install my-monitoring . --namespace monitoring
```

### 4.3 常用操作

#### 1. 查看 Chart 信息

##### 查看 Chart 元数据
```bash
# Display chart metadata
helm show chart prometheus-community/kube-prometheus-stack
```

##### 查看默认 values
```bash
# Show all default values
helm show values prometheus-community/kube-prometheus-stack

# Show first 50 lines of values (useful for large files)
helm show values prometheus-community/kube-prometheus-stack | head -50

# Save values to file for customization
helm show values prometheus-community/kube-prometheus-stack > values-default.yaml
```

##### 查看 Chart README
```bash
# Display the chart's README
helm show readme prometheus-community/kube-prometheus-stack
```

#### 2. 搜索和查询

##### 搜索仓库中的 Charts
```bash
# Search all charts in prometheus-community repository
helm search repo prometheus-community

# Search for specific chart
helm search repo prometheus-community/kube-prometheus-stack

# Show all versions of a chart
helm search repo prometheus-community/kube-prometheus-stack --versions
```

#### 3. 安装和管理

##### 基础安装
```bash
# Install with default values
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace

# Install with custom values file
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --values values.yaml
```

##### 查看已安装的 Releases
```bash
# List releases in current namespace
helm list

# List releases in all namespaces
helm list -A

# List releases in specific namespace
helm list -n monitoring
```

### 4.4 使用 --values 参数

#### `--values` 参数的基本用法说明

`--values` 参数用于指定一个或多个 YAML 文件，这些文件包含自定义的值，用于覆盖 Helm chart 默认值。它可以帮助用户在安装或升级 Chart 时进行配置自定义。

#### 单个 values 文件使用示例

要使用一个自定义的 `values.yaml` 文件来安装 Helm Chart，你可以使用以下命令：

```bash
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --values values.yaml
```

#### 多个 values 文件使用示例

你可以通过指定多个 `--values` 参数来合并多个文件，从而灵活定制配置。后指定的文件覆盖先前文件中的值：

```bash
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --values base-values.yaml \
  --values overrides.yaml
```

或者在升级时使用：

```bash
helm upgrade kube-monitor prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --values base-values.yaml \
  --values overrides.yaml
```

#### values 文件的加载顺序说明

当指定多个 `--values` 文件时，加载顺序是从左到右，即先加载早前指定的文件，然后依次加载后续文件。后加载的文件中的设置会覆盖前面文件中的同名设置。

### 4.5 使用本地 values 文件进行升级

在进行版本升级时，有时需要自定义参数配置，此时便可使用本地的 values 文件。通过提供自定义的 values 文件，可以更加灵活地控制升级过程中的配置，确保符合特定的部署需求。

#### 升级前的检查步骤

在执行 Helm 升级操作之前，建议进行以下检查步骤以确保升级的安全性和成功率：

##### 1. 检查当前 Release 状态

首先确认当前 release 的运行状态是否正常：

```bash
# Check the current release status
helm status kube-prometheus-stack -n monitoring

# View the release history to understand previous upgrades
helm history kube-prometheus-stack -n monitoring

# Get detailed information about the current release
helm get all kube-prometheus-stack -n monitoring
```

##### 2. 备份当前配置

在升级前备份当前的配置是最佳实践，以便在需要时能够回滚：

```bash
# Backup current values
helm get values kube-prometheus-stack -n monitoring > values-backup-$(date +%Y%m%d-%H%M%S).yaml

# Backup the complete release manifest
helm get manifest kube-prometheus-stack -n monitoring > manifest-backup-$(date +%Y%m%d-%H%M%S).yaml

# Optional: Backup all release information
helm get all kube-prometheus-stack -n monitoring > release-backup-$(date +%Y%m%d-%H%M%S).yaml
```

##### 3. 比较新旧 Values 文件差异

了解新版本与当前版本的配置差异对于评估升级影响至关重要：

```bash
# Get current values
helm get values kube-prometheus-stack -n monitoring > current-values.yaml

# Get default values of the new version
helm show values prometheus-community/kube-prometheus-stack --version <NEW_VERSION> > new-default-values.yaml

# Compare the differences
diff -u current-values.yaml new-default-values.yaml

# Or use a more visual diff tool if available
# vimdiff current-values.yaml new-default-values.yaml
# code --diff current-values.yaml new-default-values.yaml
```

##### 4. 验证 Values 文件语法

确保自定义的 values 文件语法正确，避免因配置错误导致升级失败：

```bash
# Validate YAML syntax
yamllint values.yaml

# Or use yq for validation
yq eval '.' values.yaml > /dev/null && echo "YAML syntax is valid"

# Perform a dry-run to check for configuration errors
helm upgrade kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --values values.yaml \
  --dry-run \
  --debug

# Use helm lint to check chart and values compatibility
helm lint prometheus-community/kube-prometheus-stack --values values.yaml
```

完成以上检查步骤后，您可以更有信心地进行实际的升级操作。

#### 升级 Release
```bash
# Upgrade to latest version
helm upgrade kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring

# Upgrade to specific version
helm upgrade kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --version 61.3.1
```

#### 卸载 Release
```bash
# Uninstall the release
helm uninstall kube-prometheus-stack --namespace monitoring
```

#### 升级命令示例

以下是一些常用的 Helm 升级命令示例：

1. **基本升级命令（使用本地 values-default.yaml）**

```bash
helm upgrade kube-monitor prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --values /Users/bhuang/code/Bhuang-k8s/helm/kube-prometheus-stack/values-default.yaml
```

2. **指定版本号的升级命令**

```bash
helm upgrade kube-monitor prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --version 61.3.1 \
  --values /Users/bhuang/code/Bhuang-k8s/helm/kube-prometheus-stack/values-default.yaml
```

3. **结合多个 values 文件的升级命令**

```bash
helm upgrade kube-monitor prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --values /Users/bhuang/code/Bhuang-k8s/helm/kube-prometheus-stack/values-default.yaml \
  --values additional-values.yaml
```

4. **使用 --reuse-values 的升级命令**

```bash
helm upgrade kube-monitor prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --reuse-values
```

### 4.6 使用 --set 和 --values 参数进行升级

在 Helm 升级过程中，`--set` 和 `--values` 参数都可以用来自定义配置值，但它们有不同的使用场景：

#### --set 参数

`--set` 用于在命令行中直接设置单个值，适合简单的配置修改：

```bash
# 设置单个值
helm upgrade kube-monitor prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set grafana.adminPassword="newpassword"

# 设置多个值
helm upgrade kube-monitor prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set grafana.adminPassword="newpassword" \
  --set prometheus.prometheusSpec.retention="30d"

# 设置嵌套值
helm upgrade kube-monitor prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage="100Gi"
```

#### 结合使用 --set 和 --values

你可以同时使用 `--values` 和 `--set` 参数，`--set` 指定的值会覆盖 `--values` 文件中的相应设置：

```bash
helm upgrade kube-monitor prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --values values.yaml \
  --set grafana.adminPassword="override-password"
```

#### 参数优先级

当同时使用多种配置方式时，优先级从低到高为：
1. Chart 默认值
2. `--values` 文件（按指定顺序）
3. `--set` 参数
4. `--set-string` 参数
5. `--set-file` 参数

#### 实际应用示例

```bash
# 使用基础配置文件，但临时覆盖某些敏感信息
helm upgrade kube-monitor prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --values base-values.yaml \
  --values env-specific-values.yaml \
  --set grafana.adminPassword="$GRAFANA_ADMIN_PASSWORD" \
  --set alertmanager.config.global.slack_api_url="$SLACK_WEBHOOK_URL"
```

### 4.7 --set 和 --values 混合使用的详细示例

在实际使用中，经常需要结合使用 `--set` 和 `--values` 参数来实现灵活的配置管理。以下是一些典型的使用场景：

#### 基础混合使用场景

##### 场景 1: 开发环境配置

假设有一个基础的 `values-dev.yaml` 文件：

```yaml
# values-dev.yaml
grafana:
  enabled: true
  replicas: 1
  resources:
    limits:
      cpu: 100m
      memory: 128Mi

prometheus:
  prometheusSpec:
    retention: 7d
    resources:
      limits:
        cpu: 500m
        memory: 1Gi
```

使用时临时调整某些参数：

```bash
# Install with dev values but override admin password
helm install kube-monitor prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --values values-dev.yaml \
  --set grafana.adminPassword="dev-temp-password" \
  --set prometheus.prometheusSpec.retention="3d"
```

##### 场景 2: 多环境部署

```bash
# Base configuration for all environments
helm install kube-monitor prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --values values-base.yaml \
  --values values-${ENVIRONMENT}.yaml \
  --set grafana.ingress.hosts[0]="grafana-${ENVIRONMENT}.example.com" \
  --set global.environment="${ENVIRONMENT}"
```

#### 复杂混合使用场景

##### 场景 3: 多层配置覆盖

假设有以下配置文件结构：
- `values-base.yaml`: 基础配置
- `values-cloud.yaml`: 云环境特定配置
- `values-prod.yaml`: 生产环境配置
- `values-secrets.yaml`: 敏感信息配置

```bash
# Complex multi-layer configuration
helm upgrade kube-monitor prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --values values-base.yaml \
  --values values-cloud.yaml \
  --values values-prod.yaml \
  --values values-secrets.yaml \
  --set prometheus.prometheusSpec.replicas=3 \
  --set alertmanager.alertmanagerSpec.replicas=3 \
  --set grafana.persistence.enabled=true \
  --set grafana.persistence.size="50Gi" \
  --set-string grafana.image.tag="10.0.3" \
  --set global.imageRegistry="private-registry.example.com"
```

##### 场景 4: 动态配置与条件部署

```bash
# Dynamic configuration based on cluster capabilities
KUBE_VERSION=$(kubectl version -o json | jq -r '.serverVersion.gitVersion')
CLUSTER_NAME=$(kubectl config current-context)
STORAGE_CLASS=$(kubectl get storageclass -o json | jq -r '.items[0].metadata.name')

helm install kube-monitor prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --values values-base.yaml \
  --values values-${CLUSTER_NAME}.yaml \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.storageClassName="${STORAGE_CLASS}" \
  --set alertmanager.alertmanagerSpec.storage.volumeClaimTemplate.spec.storageClassName="${STORAGE_CLASS}" \
  --set grafana.persistence.storageClassName="${STORAGE_CLASS}" \
  --set global.kubernetes.version="${KUBE_VERSION}" \
  --set global.cluster.name="${CLUSTER_NAME}"
```

#### 参数覆盖的实际案例

##### 案例 1: 配置文件与命令行参数的覆盖顺序

假设 `values-custom.yaml` 包含：

```yaml
# values-custom.yaml
grafana:
  adminPassword: "password-from-file"
  replicas: 2
  service:
    type: ClusterIP
    port: 80
```

执行以下命令：

```bash
helm install kube-monitor prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --values values-custom.yaml \
  --set grafana.adminPassword="password-from-cli" \
  --set grafana.service.type="NodePort" \
  --set grafana.service.nodePort=30080
```

最终生效的配置将是：
- `adminPassword`: "password-from-cli" (被 --set 覆盖)
- `replicas`: 2 (来自 values 文件)
- `service.type`: "NodePort" (被 --set 覆盖)
- `service.port`: 80 (来自 values 文件)
- `service.nodePort`: 30080 (新增的配置)

##### 案例 2: 数组和对象的覆盖行为

```bash
# Array handling example
helm install kube-monitor prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --values values.yaml \
  --set prometheus.prometheusSpec.additionalScrapeConfigs[0].job_name="custom-job" \
  --set prometheus.prometheusSpec.additionalScrapeConfigs[0].static_configs[0].targets[0]="target1:9090" \
  --set prometheus.prometheusSpec.additionalScrapeConfigs[0].static_configs[0].targets[1]="target2:9090"

# Object merging example  
helm install kube-monitor prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --values values.yaml \
  --set-json 'grafana.dashboardProviders={"dashboardproviders.yaml":{"apiVersion":1,"providers":[{"name":"default","folder":"","type":"file","options":{"path":"/var/lib/grafana/dashboards"}}]}}'
```

##### 案例 3: 处理特殊字符和复杂值

```bash
# Handling special characters
helm install kube-monitor prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --values values.yaml \
  --set alertmanager.config.route.group_by='["alertname","cluster","service"]' \
  --set-string prometheus.prometheusSpec.externalLabels.cluster="prod-us-east-1" \
  --set prometheus.prometheusSpec.retention="30d" \
  --set-file alertmanager.config.templates[0]=/path/to/custom-template.tmpl

# Using environment variables with special characters
export SLACK_URL="https://hooks.slack.com/services/XXX/YYY/ZZZ"
export DB_PASSWORD='p@ssw0rd!#$%'

helm install kube-monitor prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --values values.yaml \
  --set alertmanager.config.global.slack_api_url="${SLACK_URL}" \
  --set-string externalDatabase.password="${DB_PASSWORD}"
```

#### 最佳实践建议

1. **分层配置管理**
   - 使用 values 文件管理稳定的、环境特定的配置
   - 使用 --set 处理动态值、敏感信息或临时覆盖

2. **配置优先级规划**
   ```bash
   # Recommended order: base -> environment -> secrets -> overrides
   helm install release-name chart-name \
     --values values-base.yaml \
     --values values-${ENV}.yaml \
     --values values-secrets.yaml \
     --set image.tag="${CI_COMMIT_SHA}" \
     --set ingress.hosts[0]="${DYNAMIC_HOSTNAME}"
   ```

3. **调试配置合并结果**
   ```bash
   # Preview merged values without installing
   helm install kube-monitor prometheus-community/kube-prometheus-stack \
     --namespace monitoring \
     --values values1.yaml \
     --values values2.yaml \
     --set key1=value1 \
     --dry-run --debug 2>&1 | grep "USER-SUPPLIED VALUES:" -A 1000
   ```

### 4.8 高级用法

#### 查看特定版本的 values
```bash
# Show values for a specific version
helm show values prometheus-community/kube-prometheus-stack --version 61.3.1
```

#### 生成 Kubernetes 清单文件（不安装）
```bash
# Generate manifests without installing
helm template kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --values values.yaml > manifests.yaml
```

#### 验证 Chart
```bash
# Lint the chart
helm lint prometheus-community/kube-prometheus-stack

# Dry run installation
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --dry-run \
  --debug
```

### 4.9 自定义配置示例

创建一个 `values.yaml` 文件来自定义配置：

```yaml
# Example custom values for kube-prometheus-stack
grafana:
  enabled: true
  adminPassword: "your-secure-password"
  ingress:
    enabled: true
    hosts:
      - grafana.example.com

prometheus:
  prometheusSpec:
    retention: 30d
    storageSpec:
      volumeClaimTemplate:
        spec:
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 50Gi

alertmanager:
  alertmanagerSpec:
    storage:
      volumeClaimTemplate:
        spec:
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 10Gi
```

### 4.10 故障排查

#### 查看 Release 状态
```bash
# Get release status
helm status kube-prometheus-stack -n monitoring

# Get release history
helm history kube-prometheus-stack -n monitoring
```

#### 回滚到之前的版本
```bash
# Rollback to previous version
helm rollback kube-prometheus-stack -n monitoring

# Rollback to specific revision
helm rollback kube-prometheus-stack 2 -n monitoring
```

---

## 5. 为自己的项目创建 Helm Chart

为自己的项目创建 Helm Chart 是掌握 Helm 的重要技能。本章将通过一个完整的示例，详细说明如何从零开始为一个 Web 应用创建 Helm Chart。

### 5.1 项目概述

假设我们有一个简单的 Web 应用项目，包含：
- 一个 Node.js Web 应用
- 一个 Redis 缓存服务
- 需要支持不同环境的部署（开发、测试、生产）

### 5.2 第一步：创建基础 Chart 结构

#### 使用 Helm 命令创建基础框架

```bash
# 创建新的 chart
helm create my-web-app

# 查看创建的目录结构
tree my-web-app/
```

这将创建以下目录结构：

```
my-web-app/
├── Chart.yaml          # Chart 元数据
├── values.yaml         # 默认配置值
├── templates/          # Kubernetes 资源模板
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── serviceaccount.yaml
│   ├── hpa.yaml
│   ├── NOTES.txt
│   └── tests/
│       └── test-connection.yaml
├── charts/            # 依赖的子 charts
└── .helmignore       # 忽略文件列表
```

#### 清理不需要的文件

```bash
cd my-web-app/

# 删除不需要的文件，保留核心文件
rm templates/serviceaccount.yaml
rm templates/hpa.yaml
rm -rf templates/tests/
```

### 5.3 第二步：配置 Chart.yaml

编辑 `Chart.yaml` 文件，定义 Chart 的基本信息：

```yaml
# Chart.yaml
apiVersion: v2
name: my-web-app
description: A Helm chart for my web application
type: application
version: 0.1.0
appVersion: "1.0.0"
keywords:
  - web
  - nodejs
  - redis
home: https://github.com/yourusername/my-web-app
sources:
  - https://github.com/yourusername/my-web-app
maintainers:
  - name: Your Name
    email: your.email@example.com
dependencies:
  - name: redis
    version: "17.3.14"
    repository: "https://charts.bitnami.com/bitnami"
    condition: redis.enabled
```

### 5.4 第三步：设计 values.yaml

创建一个完整的 `values.yaml` 文件，定义所有可配置的参数：

```yaml
# values.yaml

# 全局配置
global:
  imageRegistry: ""
  storageClass: ""

# 应用配置
app:
  name: my-web-app
  version: "1.0.0"

# 镜像配置
image:
  repository: my-registry/my-web-app
  pullPolicy: IfNotPresent
  tag: ""  # 如果为空，使用 Chart.appVersion

# 镜像拉取密钥
imagePullSecrets: []

# 副本数配置
replicaCount: 1

# 服务账户配置
serviceAccount:
  create: true
  annotations: {}
  name: ""

# Pod 安全上下文
podSecurityContext:
  fsGroup: 2000

# 容器安全上下文
securityContext:
  capabilities:
    drop:
    - ALL
  readOnlyRootFilesystem: true
  runAsNonRoot: true
  runAsUser: 1000

# 服务配置
service:
  type: ClusterIP
  port: 3000
  targetPort: 3000

# Ingress 配置
ingress:
  enabled: false
  className: "nginx"
  annotations: {}
    # kubernetes.io/ingress.class: nginx
    # cert-manager.io/cluster-issuer: letsencrypt-prod
  hosts:
    - host: my-app.example.com
      paths:
        - path: /
          pathType: ImplementationSpecific
  tls: []
    # - secretName: my-app-tls
    #   hosts:
    #     - my-app.example.com

# 资源限制
resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 250m
    memory: 256Mi

# 存活性探针
livenessProbe:
  httpGet:
    path: /health
    port: http
  initialDelaySeconds: 30
  periodSeconds: 10

# 就绪性探针
readinessProbe:
  httpGet:
    path: /ready
    port: http
  initialDelaySeconds: 5
  periodSeconds: 5

# 自动扩缩容
autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80
  targetMemoryUtilizationPercentage: 80

# 节点选择器
nodeSelector: {}

# 容忍度
tolerations: []

# 亲和性
affinity: {}

# 环境变量
env:
  - name: NODE_ENV
    value: "production"
  - name: PORT
    value: "3000"

# 从 ConfigMap 获取的环境变量
envFrom: []

# 配置文件
config:
  # 应用配置文件内容
  app.json: |
    {
      "name": "my-web-app",
      "version": "1.0.0",
      "environment": "production"
    }

# 密钥
secrets:
  # 数据库连接字符串等敏感信息
  database:
    url: ""
    username: ""
    password: ""

# 持久化存储
persistence:
  enabled: false
  storageClass: ""
  accessMode: ReadWriteOnce
  size: 10Gi
  mountPath: /app/data

# Redis 配置（作为依赖）
redis:
  enabled: true
  auth:
    enabled: true
    password: "redis-password"
  master:
    persistence:
      enabled: true
      size: 8Gi

# 监控配置
monitoring:
  enabled: false
  serviceMonitor:
    enabled: false
    interval: 30s
    path: /metrics
```

### 5.5 第四步：创建 Deployment 模板

编辑 `templates/deployment.yaml`：

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "my-web-app.fullname" . }}
  labels:
    {{- include "my-web-app.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "my-web-app.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
        checksum/secret: {{ include (print $.Template.BasePath "/secret.yaml") . | sha256sum }}
      labels:
        {{- include "my-web-app.selectorLabels" . | nindent 8 }}
    spec:
      {{- with .Values.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      serviceAccountName: {{ include "my-web-app.serviceAccountName" . }}
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
          env:
            {{- range .Values.env }}
            - name: {{ .name }}
              value: {{ .value | quote }}
            {{- end }}
            # Redis 连接配置
            {{- if .Values.redis.enabled }}
            - name: REDIS_HOST
              value: {{ include "my-web-app.fullname" . }}-redis-master
            - name: REDIS_PORT
              value: "6379"
            {{- if .Values.redis.auth.enabled }}
            - name: REDIS_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: {{ include "my-web-app.fullname" . }}-redis
                  key: redis-password
            {{- end }}
            {{- end }}
          {{- if .Values.envFrom }}
          envFrom:
            {{- toYaml .Values.envFrom | nindent 12 }}
          {{- end }}
          livenessProbe:
            {{- toYaml .Values.livenessProbe | nindent 12 }}
          readinessProbe:
            {{- toYaml .Values.readinessProbe | nindent 12 }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
          volumeMounts:
            - name: config
              mountPath: /app/config
              readOnly: true
            {{- if .Values.persistence.enabled }}
            - name: data
              mountPath: {{ .Values.persistence.mountPath }}
            {{- end }}
      volumes:
        - name: config
          configMap:
            name: {{ include "my-web-app.fullname" . }}-config
        {{- if .Values.persistence.enabled }}
        - name: data
          persistentVolumeClaim:
            claimName: {{ include "my-web-app.fullname" . }}-data
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

### 5.6 第五步：创建 Service 模板

编辑 `templates/service.yaml`：

```yaml
# templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "my-web-app.fullname" . }}
  labels:
    {{- include "my-web-app.labels" . | nindent 4 }}
spec:
  type: {{ .Values.service.type }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: http
      protocol: TCP
      name: http
      {{- if and (eq .Values.service.type "NodePort") .Values.service.nodePort }}
      nodePort: {{ .Values.service.nodePort }}
      {{- end }}
  selector:
    {{- include "my-web-app.selectorLabels" . | nindent 4 }}
```

### 5.7 第六步：创建 ConfigMap 和 Secret 模板

#### ConfigMap 模板

创建 `templates/configmap.yaml`：

```yaml
# templates/configmap.yaml
{{- if .Values.config }}
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "my-web-app.fullname" . }}-config
  labels:
    {{- include "my-web-app.labels" . | nindent 4 }}
data:
  {{- range $key, $value := .Values.config }}
  {{ $key }}: |
    {{- $value | nindent 4 }}
  {{- end }}
{{- end }}
```

#### Secret 模板

创建 `templates/secret.yaml`：

```yaml
# templates/secret.yaml
{{- if .Values.secrets }}
apiVersion: v1
kind: Secret
metadata:
  name: {{ include "my-web-app.fullname" . }}-secret
  labels:
    {{- include "my-web-app.labels" . | nindent 4 }}
type: Opaque
data:
  {{- range $key, $value := .Values.secrets }}
  {{- range $subkey, $subvalue := $value }}
  {{ $key }}-{{ $subkey }}: {{ $subvalue | b64enc }}
  {{- end }}
  {{- end }}
{{- end }}
```

### 5.8 第七步：创建 ServiceAccount 模板

创建 `templates/serviceaccount.yaml`：

```yaml
# templates/serviceaccount.yaml
{{- if .Values.serviceAccount.create -}}
apiVersion: v1
kind: ServiceAccount
metadata:
  name: {{ include "my-web-app.serviceAccountName" . }}
  labels:
    {{- include "my-web-app.labels" . | nindent 4 }}
  {{- with .Values.serviceAccount.annotations }}
  annotations:
    {{- toYaml . | nindent 4 }}
  {{- end }}
{{- end }}
```

### 5.9 第八步：创建 PersistentVolumeClaim 模板

创建 `templates/pvc.yaml`：

```yaml
# templates/pvc.yaml
{{- if .Values.persistence.enabled }}
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: {{ include "my-web-app.fullname" . }}-data
  labels:
    {{- include "my-web-app.labels" . | nindent 4 }}
spec:
  accessModes:
    - {{ .Values.persistence.accessMode }}
  resources:
    requests:
      storage: {{ .Values.persistence.size }}
  {{- if .Values.persistence.storageClass }}
  {{- if (eq "-" .Values.persistence.storageClass) }}
  storageClassName: ""
  {{- else }}
  storageClassName: {{ .Values.persistence.storageClass }}
  {{- end }}
  {{- end }}
{{- end }}
```

### 5.10 第九步：创建 HPA 模板

创建 `templates/hpa.yaml`：

```yaml
# templates/hpa.yaml
{{- if .Values.autoscaling.enabled }}
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: {{ include "my-web-app.fullname" . }}
  labels:
    {{- include "my-web-app.labels" . | nindent 4 }}
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: {{ include "my-web-app.fullname" . }}
  minReplicas: {{ .Values.autoscaling.minReplicas }}
  maxReplicas: {{ .Values.autoscaling.maxReplicas }}
  metrics:
    {{- if .Values.autoscaling.targetCPUUtilizationPercentage }}
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: {{ .Values.autoscaling.targetCPUUtilizationPercentage }}
    {{- end }}
    {{- if .Values.autoscaling.targetMemoryUtilizationPercentage }}
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: {{ .Values.autoscaling.targetMemoryUtilizationPercentage }}
    {{- end }}
{{- end }}
```

### 5.11 第十步：更新 _helpers.tpl

编辑 `templates/_helpers.tpl` 文件，添加常用的模板函数：

```yaml
{{/*
Expand the name of the chart.
*/}}
{{- define "my-web-app.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Create a default fully qualified app name.
*/}}
{{- define "my-web-app.fullname" -}}
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
{{- define "my-web-app.chart" -}}
{{- printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Common labels
*/}}
{{- define "my-web-app.labels" -}}
helm.sh/chart: {{ include "my-web-app.chart" . }}
{{ include "my-web-app.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/*
Selector labels
*/}}
{{- define "my-web-app.selectorLabels" -}}
app.kubernetes.io/name: {{ include "my-web-app.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}

{{/*
Create the name of the service account to use
*/}}
{{- define "my-web-app.serviceAccountName" -}}
{{- if .Values.serviceAccount.create }}
{{- default (include "my-web-app.fullname" .) .Values.serviceAccount.name }}
{{- else }}
{{- default "default" .Values.serviceAccount.name }}
{{- end }}
{{- end }}
```

### 5.12 第十一步：创建环境特定的 Values 文件

#### 开发环境配置

创建 `values-dev.yaml`：

```yaml
# values-dev.yaml
replicaCount: 1

image:
  tag: "dev-latest"

service:
  type: NodePort
  nodePort: 30080

ingress:
  enabled: true
  hosts:
    - host: my-app-dev.local
      paths:
        - path: /
          pathType: ImplementationSpecific

resources:
  limits:
    cpu: 200m
    memory: 256Mi
  requests:
    cpu: 100m
    memory: 128Mi

env:
  - name: NODE_ENV
    value: "development"
  - name: DEBUG
    value: "true"

redis:
  enabled: true
  auth:
    enabled: false
  master:
    persistence:
      enabled: false
```

#### 生产环境配置

创建 `values-prod.yaml`：

```yaml
# values-prod.yaml
replicaCount: 3

image:
  tag: "1.0.0"

service:
  type: ClusterIP

ingress:
  enabled: true
  className: "nginx"
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/rate-limit: "100"
  hosts:
    - host: my-app.example.com
      paths:
        - path: /
          pathType: ImplementationSpecific
  tls:
    - secretName: my-app-tls
      hosts:
        - my-app.example.com

resources:
  limits:
    cpu: 1000m
    memory: 1Gi
  requests:
    cpu: 500m
    memory: 512Mi

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70

persistence:
  enabled: true
  size: 20Gi

redis:
  enabled: true
  auth:
    enabled: true
    password: "secure-redis-password"
  master:
    persistence:
      enabled: true
      size: 20Gi
```

### 5.13 第十二步：更新依赖

```bash
# 添加 Redis 依赖
helm dependency update

# 验证依赖已下载
ls charts/
```

### 5.14 第十三步：测试和验证

#### 1. 语法检查

```bash
# 检查 Chart 语法
helm lint my-web-app/

# 如果有错误，会显示详细信息
```

#### 2. 模板渲染测试

```bash
# 渲染开发环境模板
helm template my-app ./my-web-app -f ./my-web-app/values-dev.yaml

# 渲染生产环境模板
helm template my-app ./my-web-app -f ./my-web-app/values-prod.yaml

# 保存渲染结果到文件
helm template my-app ./my-web-app -f ./my-web-app/values-prod.yaml > rendered-manifests.yaml
```

#### 3. 干运行测试

```bash
# 开发环境干运行
helm install my-app-dev ./my-web-app \
  -f ./my-web-app/values-dev.yaml \
  --namespace dev \
  --create-namespace \
  --dry-run --debug

# 生产环境干运行
helm install my-app-prod ./my-web-app \
  -f ./my-web-app/values-prod.yaml \
  --namespace production \
  --create-namespace \
  --dry-run --debug
```

### 5.15 第十四步：实际部署

#### 部署到开发环境

```bash
# 创建开发环境的 namespace
kubectl create namespace dev

# 部署到开发环境
helm install my-app-dev ./my-web-app \
  -f ./my-web-app/values-dev.yaml \
  --namespace dev

# 检查部署状态
helm status my-app-dev -n dev
kubectl get all -n dev
```

#### 部署到生产环境

```bash
# 创建生产环境的 namespace
kubectl create namespace production

# 部署到生产环境
helm install my-app-prod ./my-web-app \
  -f ./my-web-app/values-prod.yaml \
  --namespace production

# 检查部署状态
helm status my-app-prod -n production
kubectl get all -n production
```

### 5.16 第十五步：打包和分发

#### 打包 Chart

```bash
# 打包 Chart
helm package my-web-app/

# 这会生成 my-web-app-0.1.0.tgz 文件
```

#### 创建 Chart 仓库

```bash
# 创建本地仓库索引
helm repo index . --url http://my-charts.example.com

# 上传到 Chart 仓库（例如 Harbor、ChartMuseum 等）
# 具体步骤依赖于你使用的仓库类型
```

### 5.17 日常维护操作

#### 升级应用

```bash
# 升级开发环境
helm upgrade my-app-dev ./my-web-app \
  -f ./my-web-app/values-dev.yaml \
  --namespace dev

# 升级生产环境
helm upgrade my-app-prod ./my-web-app \
  -f ./my-web-app/values-prod.yaml \
  --namespace production
```

#### 回滚操作

```bash
# 查看发布历史
helm history my-app-prod -n production

# 回滚到上一个版本
helm rollback my-app-prod -n production

# 回滚到指定版本
helm rollback my-app-prod 2 -n production
```

#### 卸载应用

```bash
# 卸载开发环境
helm uninstall my-app-dev -n dev

# 卸载生产环境
helm uninstall my-app-prod -n production
```

### 5.18 最佳实践总结

#### 1. Chart 设计原则

- **参数化所有可配置项**：镜像标签、副本数、资源限制等
- **提供合理的默认值**：确保 Chart 可以直接使用
- **支持多环境配置**：通过不同的 values 文件
- **遵循 Kubernetes 最佳实践**：安全上下文、探针、资源限制等

#### 2. 模板编写技巧

- **使用条件语句**：`{{- if .Values.feature.enabled }}`
- **循环处理配置**：`{{- range .Values.env }}`
- **计算资源校验和**：确保配置变更时重启 Pod
- **使用 _helpers.tpl**：避免重复代码

#### 3. 安全考虑

- **不在 values.yaml 中存储敏感信息**
- **使用 Kubernetes Secrets**
- **设置适当的安全上下文**
- **启用网络策略**（如果需要）

#### 4. 测试策略

- **语法检查**：`helm lint`
- **模板渲染**：`helm template`
- **干运行**：`--dry-run --debug`
- **逐步部署**：先开发后生产

#### 5. 版本管理

- **遵循语义化版本**：Chart 版本和应用版本分开管理
- **维护变更日志**：记录每个版本的变化
- **标签策略**：为 Git 仓库打标签

通过以上步骤，你就可以为自己的项目创建一个功能完整、生产就绪的 Helm Chart。这个 Chart 支持多环境部署、自动扩缩容、持久化存储、监控集成等特性，可以作为实际项目的模板使用。

---

## 6. 高级特性和最佳实践

### 5.1 高级技巧

#### 1. 使用多个 Values 文件

```bash
# Base values + environment-specific values
helm install my-app ./my-chart \
  -f values.yaml \
  -f values-prod.yaml
```

#### 2. 模板调试

```bash
# Render templates locally
helm template my-release ./my-chart

# Debug template rendering
helm template my-release ./my-chart --debug

# Lint your chart
helm lint ./my-chart
```

#### 3. 依赖管理

```yaml
# Chart.yaml
dependencies:
  - name: postgresql
    version: "11.x.x"
    repository: "https://charts.bitnami.com/bitnami"
    condition: postgresql.enabled
```

```bash
# Update dependencies
helm dependency update ./my-chart
```

#### 4. Hooks 使用

```yaml
# Pre-install job example
apiVersion: batch/v1
kind: Job
metadata:
  name: "{{ .Release.Name }}-preinstall"
  annotations:
    "helm.sh/hook": pre-install
    "helm.sh/hook-weight": "5"
    "helm.sh/hook-delete-policy": before-hook-creation
```

### 5.2 最佳实践

1. **版本控制**
   - 始终为 Chart 和应用程序使用语义化版本
   - 在 Chart.yaml 中明确指定依赖版本

2. **安全性**
   - 不要在 values.yaml 中硬编码敏感信息
   - 使用 Kubernetes Secrets 或外部密钥管理系统

3. **可维护性**
   - 为所有可配置的值提供合理的默认值
   - 在 values.yaml 中添加详细的注释
   - 使用 `_helpers.tpl` 避免重复代码

4. **测试**
   - 使用 `helm lint` 检查 Chart 语法
   - 使用 `helm template` 验证渲染结果
   - 在部署前使用 `--dry-run` 选项

### 5.3 Helm 优势总结

Helm 通过以下方式极大地提升了 Kubernetes 应用管理的效率：

1. **降低复杂度**：将复杂的 Kubernetes 资源管理简化为简单的命令
2. **提高可靠性**：通过版本控制和回滚机制降低部署风险
3. **增强可维护性**：参数化配置使得应用更容易维护和更新
4. **促进协作**：标准化的打包格式便于团队协作和应用分享
5. **加速部署**：预构建的 Chart 和自动化流程大大缩短部署时间

这些优势使得 Helm 成为 Kubernetes 生态系统中不可或缺的工具，特别是在管理复杂应用和多环境部署时。

---

## 🔗 有用的资源

- [Helm 官方文档](https://helm.sh/docs/)
- [Artifact Hub](https://artifacthub.io/) - 查找和分享 Helm Charts
- [Helm Best Practices](https://helm.sh/docs/chart_best_practices/)
- [Chart Development Tips and Tricks](https://helm.sh/docs/howto/charts_tips_and_tricks/)
- [Prometheus Community Helm Charts](https://github.com/prometheus-community/helm-charts)
- [Kube-Prometheus-Stack Documentation](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)
- [Prometheus Operator](https://github.com/prometheus-operator/prometheus-operator)

---

## 🎯 总结

Helm 不仅仅是一个工具，它是我们在 Kubernetes 海洋中的指南针、地图和舵手。有了它，我们可以：

- 📦 **打包应用**：将复杂的应用打包成可重用的 Chart
- 🔄 **版本控制**：轻松管理应用的不同版本
- 🎯 **精确部署**：确保每次部署都是可预测和可重复的
- ⚡ **快速回滚**：在出现问题时迅速恢复
- 🔧 **配置管理**：优雅地处理不同环境的配置差异

就像古代的航海家依靠北极星导航，现代的 Kubernetes 工程师可以依靠 Helm 来确保他们的应用安全到达目的地。

从混乱到有序，从恐惧到自信，Helm 真正成为了我们在 Kubernetes 世界中不可或缺的伙伴。

---

*Happy Helming! 🚢*