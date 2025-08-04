# Helm - Kubernetes 包管理器学习指南

## 📚 目录结构

```
helm/
├── README.md                 # 本文档
├── helm-story-opening.md     # Helm 实战故事：一个团队的转型之旅
├── helm-advantages.md        # Helm 的核心优势详解
└── charts/                   # 示例 Chart 目录（待创建）
```

## 🎯 学习目标

本目录包含了 Helm 的学习资料和实践示例，帮助你：

1. **理解 Helm 的价值**：通过真实故事了解为什么需要 Helm
2. **掌握核心概念**：学习 Chart、Release、Repository 等核心概念
3. **实战操作技能**：掌握 Helm 的安装、部署、升级、回滚等操作
4. **最佳实践**：了解在生产环境中使用 Helm 的最佳实践

## 📖 阅读顺序

建议按以下顺序阅读：

1. **[helm-story-opening.md](./helm-story-opening.md)** - 从一个真实的故事开始，理解没有 Helm 时的痛点
2. **[helm-advantages.md](./helm-advantages.md)** - 深入了解 Helm 的核心优势和功能
3. **本 README 的快速入门部分** - 动手实践基本操作

## 🚀 快速入门

### 安装 Helm

```bash
# macOS
brew install helm

# Linux
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# 验证安装
helm version
```

### 基本命令

#### 1. 仓库管理

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

#### 2. Chart 操作

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

#### 3. 升级和回滚

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

## 📊 查看 Helm Release 的配置

当你需要查看已部署的 Helm Release 的配置时，可以使用以下命令：

### 查看当前 Values

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

### 查看完整的 Release 信息

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

### 比较配置差异

```bash
# Compare with default values
helm get values monitoring -n monitoring > current-values.yaml
helm show values prometheus-community/kube-prometheus-stack > default-values.yaml
diff default-values.yaml current-values.yaml
```

## 🎨 创建自定义 Chart

### Chart 结构

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

### 示例 values.yaml

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

## 🔧 高级技巧

### 1. 使用多个 Values 文件

```bash
# Base values + environment-specific values
helm install my-app ./my-chart \
  -f values.yaml \
  -f values-prod.yaml
```

### 2. 模板调试

```bash
# Render templates locally
helm template my-release ./my-chart

# Debug template rendering
helm template my-release ./my-chart --debug

# Lint your chart
helm lint ./my-chart
```

### 3. 依赖管理

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

### 4. Hooks 使用

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

## 📋 最佳实践

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

## 🔗 有用的资源

- [Helm 官方文档](https://helm.sh/docs/)
- [Artifact Hub](https://artifacthub.io/) - 查找和分享 Helm Charts
- [Helm Best Practices](https://helm.sh/docs/chart_best_practices/)
- [Chart Development Tips and Tricks](https://helm.sh/docs/howto/charts_tips_and_tricks/)

## 🎯 下一步

1. 阅读提供的故事和优势文档，深入理解 Helm 的价值
2. 动手创建你的第一个 Chart
3. 尝试部署一个复杂的应用（如 WordPress + MySQL）
4. 探索 Helm 的高级功能，如 Chart 测试和插件系统

---

*Happy Helming! 🚢*
