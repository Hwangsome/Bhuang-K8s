# Kube-Prometheus-Stack Helm Chart 使用指南

本文档记录了 `prometheus-community/kube-prometheus-stack` Helm Chart 的常用操作。

## Chart 信息说明

`prometheus-community/kube-prometheus-stack` 格式说明：
- **prometheus-community**: Helm 仓库名称
- **kube-prometheus-stack**: Chart 名称

## 前置准备

### 1. 添加 Helm 仓库

```bash
# Add the prometheus-community repository
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

# Update the repository list
helm repo update
```

### 2. 查看已添加的仓库

```bash
# List all configured repositories
helm repo list
```

## 本地 Chart 安装

除了从远程仓库拉取 Helm Chart 进行安装，还可以使用本地的 Chart 目录进行安装。这种方式在以下场景中特别有用：
- 开发和测试自定义 Chart
- 在没有网络连接的环境中部署
- 需要对官方 Chart 进行修改后使用
- 管理私有的 Chart 集合

### 基本用法

对于一个位于本地目录（如 `./mysql/`）中的 Chart，可以使用以下命令进行安装：

```bash
# Basic local chart installation
helm install mysql-local ./mysql/

# Install with namespace
helm install mysql-local ./mysql/ --namespace database --create-namespace

# Install with custom values
helm install mysql-local ./mysql/ --values my-values.yaml
```

### 本地 Chart 目录结构

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

### 本地 Chart 的常用操作

#### 1. 验证本地 Chart

在安装前，建议先验证 Chart 的正确性：

```bash
# Lint the local chart
helm lint ./mysql/

# Dry run to test the chart
helm install mysql-local ./mysql/ --dry-run --debug
```

#### 2. 查看本地 Chart 信息

```bash
# Show chart metadata
helm show chart ./mysql/

# Show default values
helm show values ./mysql/

# Show README
helm show readme ./mysql/
```

#### 3. 打包本地 Chart

如果需要分发或存档，可以将 Chart 打包：

```bash
# Package the chart
helm package ./mysql/

# Package with specific version
helm package ./mysql/ --version 1.2.3

# Package to specific directory
helm package ./mysql/ --destination ./packages/
```

#### 4. 从打包文件安装

```bash
# Install from packaged chart
helm install mysql-local mysql-1.2.3.tgz

# Install from URL
helm install mysql-local https://example.com/charts/mysql-1.2.3.tgz
```

### 本地开发工作流示例

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

### 本地 Chart 与远程仓库的结合使用

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

## 常用操作

### 1. 查看 Chart 信息

#### 查看 Chart 元数据
```bash
# Display chart metadata
helm show chart prometheus-community/kube-prometheus-stack
```

#### 查看默认 values
```bash
# Show all default values
helm show values prometheus-community/kube-prometheus-stack

# Show first 50 lines of values (useful for large files)
helm show values prometheus-community/kube-prometheus-stack | head -50

# Save values to file for customization
helm show values prometheus-community/kube-prometheus-stack > values-default.yaml
```

#### 查看 Chart README
```bash
# Display the chart's README
helm show readme prometheus-community/kube-prometheus-stack
```

### 2. 搜索和查询

#### 搜索仓库中的 Charts
```bash
# Search all charts in prometheus-community repository
helm search repo prometheus-community

# Search for specific chart
helm search repo prometheus-community/kube-prometheus-stack

# Show all versions of a chart
helm search repo prometheus-community/kube-prometheus-stack --versions
```

### 3. 安装和管理

#### 基础安装
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

#### 查看已安装的 Releases
```bash
# List releases in current namespace
helm list

# List releases in all namespaces
helm list -A

# List releases in specific namespace
helm list -n monitoring
```

### 使用 --values 参数

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

## 使用本地 values 文件进行升级

在进行版本升级时，有时需要自定义参数配置，此时便可使用本地的 values 文件。通过提供自定义的 values 文件，可以更加灵活地控制升级过程中的配置，确保符合特定的部署需求。

### 升级前的检查步骤

在执行 Helm 升级操作之前，建议进行以下检查步骤以确保升级的安全性和成功率：

#### 1. 检查当前 Release 状态

首先确认当前 release 的运行状态是否正常：

```bash
# Check the current release status
helm status kube-prometheus-stack -n monitoring

# View the release history to understand previous upgrades
helm history kube-prometheus-stack -n monitoring

# Get detailed information about the current release
helm get all kube-prometheus-stack -n monitoring
```

#### 2. 备份当前配置

在升级前备份当前的配置是最佳实践，以便在需要时能够回滚：

```bash
# Backup current values
helm get values kube-prometheus-stack -n monitoring > values-backup-$(date +%Y%m%d-%H%M%S).yaml

# Backup the complete release manifest
helm get manifest kube-prometheus-stack -n monitoring > manifest-backup-$(date +%Y%m%d-%H%M%S).yaml

# Optional: Backup all release information
helm get all kube-prometheus-stack -n monitoring > release-backup-$(date +%Y%m%d-%H%M%S).yaml
```

#### 3. 比较新旧 Values 文件差异

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

#### 4. 验证 Values 文件语法

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

### 升级命令示例

以下是一些常用的 Helm 升级命令示例，使用您提供的参数:

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

### 使用 --set 和 --values 参数进行升级

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

### 5. --set 和 --values 混合使用的详细示例

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

### 4. 高级用法

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

## 自定义配置示例

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

## 故障排查

### 查看 Release 状态
```bash
# Get release status
helm status kube-prometheus-stack -n monitoring

# Get release history
helm history kube-prometheus-stack -n monitoring
```

### 回滚到之前的版本
```bash
# Rollback to previous version
helm rollback kube-prometheus-stack -n monitoring

# Rollback to specific revision
helm rollback kube-prometheus-stack 2 -n monitoring
```

## 参考链接

- [Prometheus Community Helm Charts](https://github.com/prometheus-community/helm-charts)
- [Kube-Prometheus-Stack Documentation](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)
- [Prometheus Operator](https://github.com/prometheus-operator/prometheus-operator)
