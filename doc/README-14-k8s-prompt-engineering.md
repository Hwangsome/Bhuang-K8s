# Kubernetes 提示工程最佳实践

## 1. 概述

本章节介绍如何使用提示工程技术来高效管理和操作 Kubernetes 集群。通过优化提示词设计，可以让 AI 助手更准确地生成 Kubernetes 配置文件、解决故障和优化集群性能。

## 2. 基础提示模板

### 2.1 资源创建模板

创建 Kubernetes 资源时的标准提示模板：

```yaml
# Prompt template for creating Kubernetes resources
apiVersion: {api_version}
kind: {resource_type}
metadata:
  name: {resource_name}
  namespace: {namespace}
  labels:
    app: {app_name}
    tier: {tier}
    environment: {env}
spec:
  # Specific configuration based on resource type
```

示例提示词：
```
创建一个 Nginx Deployment，包含以下要求：
- 3个副本
- 使用 nginx:1.21 镜像
- 配置健康检查
- 设置资源限制（CPU: 100m-500m, Memory: 128Mi-512Mi）
- 添加环境变量 NGINX_PORT=8080
```

### 2.2 故障排查模板

用于排查 Kubernetes 问题的系统提示：

```
我需要排查 Kubernetes 集群中的问题。请按以下步骤操作：
1. 检查 Pod 状态：kubectl get pods -n {namespace}
2. 查看 Pod 日志：kubectl logs {pod_name} -n {namespace}
3. 描述 Pod 详情：kubectl describe pod {pod_name} -n {namespace}
4. 检查事件：kubectl get events -n {namespace}
5. 验证资源配置：kubectl get {resource_type} {resource_name} -o yaml
```

## 3. 高级提示工程技巧

### 3.1 上下文增强

为 AI 提供充分的上下文信息以生成更准确的配置：

```
背景信息：
- 集群版本：Kubernetes 1.28
- 云平台：AWS EKS
- 网络插件：Calico
- 存储类：gp3-csi
- 监控方案：Prometheus + Grafana

任务：创建一个高可用的 PostgreSQL StatefulSet 配置
```

### 3.2 链式思考（Chain of Thought）

引导 AI 逐步思考和解决复杂问题：

```
让我们逐步设计一个微服务部署方案：
1. 首先，定义服务的基本需求（CPU、内存、存储）
2. 接着，设计 Service 和 Ingress 配置
3. 然后，考虑持久化存储需求
4. 最后，添加监控和日志收集配置
```

### 3.3 Few-shot 学习

提供示例帮助 AI 理解期望的输出格式：

```
示例1：创建简单的 Web 应用部署
输入：部署一个 Python Flask 应用
输出：[完整的 Deployment YAML]

示例2：创建带数据库的应用
输入：部署 WordPress + MySQL
输出：[完整的 Deployment + Service + PVC YAML]

现在，请创建：部署一个 Spring Boot 应用with Redis 缓存
```

## 4. 特定场景的提示模板

### 4.1 安全配置

```
创建安全的 Kubernetes 配置，包含：
- Pod Security Standards (restricted)
- NetworkPolicy 限制
- RBAC 权限配置
- Secret 管理
- 镜像扫描策略
```

### 4.2 性能优化

```
优化以下 Kubernetes 资源配置的性能：
[粘贴现有配置]

重点关注：
- HPA/VPA 配置
- 资源请求和限制
- 节点亲和性
- Pod 反亲和性
- 预热和优雅关闭
```

### 4.3 多环境部署

```
基于以下基础配置，生成多环境部署文件：
- 开发环境：1个副本，较小资源
- 测试环境：2个副本，中等资源
- 生产环境：3个副本，高资源，高可用配置

使用 Kustomize overlays 结构组织
```

## 5. 提示工程最佳实践

### 5.1 明确性原则

- ✅ 好的提示：
  ```
  创建一个 ConfigMap，名称为 app-config，包含数据库连接字符串（不包含密码）和应用端口配置
  ```

- ❌ 差的提示：
  ```
  创建配置
  ```

### 5.2 结构化输入

使用结构化格式提供输入信息：

```yaml
service_requirements:
  name: user-service
  type: microservice
  tech_stack: java-spring-boot
  dependencies:
    - database: postgresql
    - cache: redis
    - message_queue: rabbitmq
  scaling:
    min_replicas: 2
    max_replicas: 10
    target_cpu: 70
  resources:
    requests:
      cpu: 500m
      memory: 1Gi
    limits:
      cpu: 2000m
      memory: 4Gi
```

### 5.3 迭代优化

通过迭代改进提示词质量：

```
# 第一次迭代
创建 Deployment

# 第二次迭代  
创建一个 Nginx Deployment with 3 replicas

# 第三次迭代
创建一个生产级别的 Nginx Deployment：
- 3个副本，跨可用区分布
- 配置就绪探针和存活探针
- 设置合适的资源限制
- 配置 PodDisruptionBudget
- 添加必要的标签和注解
```

## 6. 集成 AI 工具链

### 6.1 kubectl AI 插件

配置 kubectl 插件以支持 AI 驱动的操作：

```bash
# Install kubectl-ai plugin
kubectl krew install ai

# Configure AI backend
kubectl ai config set provider openai
kubectl ai config set model gpt-4

# Use AI-powered commands
kubectl ai "show me all failing pods"
kubectl ai "create a service for my nginx deployment"
```

### 6.2 GitOps 集成

在 GitOps 工作流中使用 AI 生成和验证配置：

```yaml
# .github/workflows/ai-review.yml
name: AI Configuration Review
on:
  pull_request:
    paths:
      - 'k8s/**.yaml'
      
jobs:
  ai-review:
    runs-on: ubuntu-latest
    steps:
      - name: AI Review Kubernetes Configs
        run: |
          # AI reviews configuration for best practices
          ai-k8s-reviewer check --files k8s/
```

## 7. 常见陷阱和解决方案

### 7.1 避免歧义

❌ 歧义提示：
```
部署应用到集群
```

✅ 明确提示：
```
部署 myapp:v1.2.3 镜像到 production 命名空间，使用现有的 myapp-configmap 配置
```

### 7.2 版本兼容性

始终指定 Kubernetes API 版本：

```
使用 Kubernetes 1.28 兼容的 API 版本创建以下资源：
- Deployment (apps/v1)
- Service (v1)  
- Ingress (networking.k8s.io/v1)
```

### 7.3 安全考虑

在提示中明确安全要求：

```
创建 Deployment 时请确保：
- 不以 root 用户运行
- 设置 readOnlyRootFilesystem: true
- 禁用特权模式
- 设置 securityContext
```

## 8. 实战案例

### 8.1 完整应用部署

```
项目需求：
部署一个电商应用，包含：
- 前端：React SPA
- API：Node.js Express
- 数据库：PostgreSQL 主从复制
- 缓存：Redis Sentinel
- 消息队列：RabbitMQ 集群
- 监控：集成 Prometheus 指标

请生成完整的 Kubernetes 配置，包括：
1. 所有服务的 Deployment/StatefulSet
2. Service 和 Ingress 配置
3. ConfigMap 和 Secret 管理
4. PVC 存储配置
5. NetworkPolicy 安全策略
6. 监控和告警规则
```

### 8.2 故障恢复场景

```
场景：生产环境 PostgreSQL 主节点故障

请提供：
1. 故障检测命令
2. 手动故障转移步骤
3. 数据一致性验证
4. 应用重连配置
5. 事后分析建议
```

## 9. 提示模板库

### 9.1 基础设施类

```yaml
templates:
  - name: "create-stateful-database"
    prompt: |
      创建一个生产级别的 {database_type} StatefulSet：
      - 版本：{version}
      - 副本数：{replicas}
      - 存储大小：{storage_size}
      - 备份策略：{backup_policy}
      
  - name: "setup-monitoring"  
    prompt: |
      为 {service_name} 服务配置完整的监控方案：
      - Prometheus 指标收集
      - Grafana 仪表板
      - 告警规则（CPU、内存、错误率）
```

### 9.2 运维操作类

```yaml
operations:
  - name: "rolling-update"
    prompt: |
      执行 {deployment_name} 的滚动更新：
      - 新镜像：{new_image}
      - 更新策略：{strategy}
      - 健康检查等待时间：{health_check_delay}
      
  - name: "scale-application"
    prompt: |
      基于以下指标扩容 {app_name}：
      - 当前 CPU 使用率：{current_cpu}%
      - 当前内存使用率：{current_memory}%
      - 当前请求延迟：{current_latency}ms
      - 预期流量增长：{expected_traffic_increase}%
```

## 10. 总结

提示工程在 Kubernetes 运维中的应用可以显著提高工作效率。关键要点：

1. **明确性**：提供清晰、具体的需求描述
2. **上下文**：包含足够的背景信息
3. **结构化**：使用结构化格式组织输入
4. **示例驱动**：通过示例引导期望输出
5. **迭代优化**：持续改进提示词质量
6. **安全第一**：始终考虑安全最佳实践
7. **版本兼容**：注意 Kubernetes API 版本差异

通过掌握这些提示工程技巧，可以让 AI 成为 Kubernetes 运维的得力助手，加速开发和部署流程，提高系统可靠性。
