# Kubernetes Deployment 深度指南：从理论到实践

## 📚 目录

### 第一部分：基础概念
1. [简介](#简介)
2. [为什么需要 Deployment？](#为什么需要-deployment)
3. [Deployment 的核心概念](#deployment-的核心概念)
4. [Deployment vs ReplicaSet vs Pod](#deployment-vs-replicaset-vs-pod)

### 第二部分：实战操作
5. [安装与配置](#安装与配置)
6. [创建第一个 Deployment](#创建第一个-deployment)
7. [更新策略详解](#更新策略详解)
8. [回滚机制](#回滚机制)
9. [扩缩容操作](#扩缩容操作)

### 第三部分：进阶技巧
10. [生产环境最佳实践](#生产环境最佳实践)
11. [监控与日志](#监控与日志)
12. [故障排查](#故障排查)
13. [性能优化](#性能优化)

### 第四部分：案例与总结
14. [真实案例分析](#真实案例分析)
15. [常见问题 FAQ](#常见问题-faq)
16. [总结与展望](#总结与展望)

---

## 简介

Kubernetes Deployment 是容器编排领域的核心组件，它不仅简化了应用的部署流程，更是实现了声明式管理的理念。本指南将从实际场景出发，深入浅出地介绍 Deployment 的方方面面。

## 为什么需要 Deployment？

### 🎭 一个真实的故事：从混乱到有序

#### 第一幕：ReplicaSet 时代的痛苦

时间回到 2019 年的一个深夜，某电商公司的运维工程师小李正在处理一个紧急的版本更新。公司的主要应用运行在 Kubernetes 集群上，使用 ReplicaSet 管理着 20 个 Pod 副本。

"又要更新了..." 小李深吸一口气，开始了他的"手工艺术"：

```bash
# Step 1: Create new ReplicaSet with new image version
kubectl create -f app-replicaset-v2.yaml

# Step 2: Manually scale down old ReplicaSet
kubectl scale replicaset app-v1 --replicas=15
kubectl scale replicaset app-v2 --replicas=5

# Step 3: Wait and check...
sleep 30
kubectl get pods | grep app

# Step 4: Continue scaling...
kubectl scale replicaset app-v1 --replicas=10
kubectl scale replicaset app-v2 --replicas=10

# ... repeat until complete
```

凌晨 2 点，当小李执行到第 15 步时，一个失误发生了——他错误地将旧版本的副本数设置为 0，而新版本只有 10 个副本在运行。瞬间，50% 的流量无法处理，告警电话响起...

#### 第二幕：问题的根源

使用 ReplicaSet 管理应用时，主要面临以下挑战：

1. **手动管理版本更新**
   - 需要创建新的 ReplicaSet
   - 手动调整新旧版本的副本数
   - 容易出现人为失误

2. **缺乏更新策略**
   - 无法定义滚动更新的速度
   - 没有自动的健康检查机制
   - 更新失败时无法自动回滚

3. **版本管理混乱**
   - 多个 ReplicaSet 同时存在
   - 难以追踪版本历史
   - 回滚操作复杂

#### 第三幕：Deployment 的救赎

经历了那次事故后，小李开始研究 Deployment。当他第一次使用 Deployment 更新应用时，简直不敢相信自己的眼睛：

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-deployment
spec:
  replicas: 20
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 5        # Maximum pods above desired number
      maxUnavailable: 2  # Maximum pods that can be unavailable
  template:
    spec:
      containers:
      - name: app
        image: myapp:v2.0  # Just change this line!
```

```bash
# One command to rule them all
kubectl apply -f deployment.yaml
```

就这样，Kubernetes 自动完成了整个更新过程：
- ✅ 自动创建新的 ReplicaSet
- ✅ 按照定义的策略逐步更新 Pod
- ✅ 持续进行健康检查
- ✅ 保留历史版本，支持一键回滚

### 📊 对比：有无 Deployment 的差异

| 特性 | 仅使用 ReplicaSet | 使用 Deployment |
|------|-------------------|------------------|
| **版本更新** | 手动创建新 RS，逐步调整副本数 | 自动滚动更新 |
| **更新时间** | 30-60 分钟（人工操作） | 5-10 分钟（自动完成） |
| **错误率** | 高（人为失误） | 低（自动化流程） |
| **回滚操作** | 复杂，需要手动恢复 | 一条命令即可回滚 |
| **版本历史** | 需要手动管理 | 自动保留历史记录 |
| **更新策略** | 无 | 支持多种策略 |
| **健康检查** | 需要手动验证 | 自动健康检查 |
| **声明式管理** | 不支持 | 完全支持 |

### 🚀 演进的必要性

Deployment 的出现不是偶然，而是 Kubernetes 生态演进的必然结果：

1. **从命令式到声明式**
   - ReplicaSet：告诉 K8s "怎么做"
   - Deployment：告诉 K8s "要什么"

2. **从手动到自动**
   - 减少人为干预
   - 提高部署效率
   - 降低操作风险

3. **从简单到智能**
   - 内置更新策略
   - 自动版本管理
   - 智能健康检查

正如小李在事后总结中写道："Deployment 不仅仅是一个工具，它代表了一种理念——让机器做机器擅长的事，让人专注于更有价值的工作。"

## Deployment 概述

### 📖 定义

Deployment 是 Kubernetes 中用于声明式管理应用部署的 API 对象。它提供了对 Pod 和 ReplicaSet 的声明式更新能力，是运行无状态应用的推荐方式。

```yaml
# A simple Deployment definition
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
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
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```

### 🎯 主要功能

1. **声明式更新**
   - 用户只需要描述期望状态
   - Kubernetes 自动实现状态转换
   - 支持配置文件版本管理

2. **自动化部署管理**
   - 自动创建和管理 ReplicaSet
   - 智能协调 Pod 的创建、更新和删除
   - 内置健康检查和自愈能力

3. **版本控制与回滚**
   - 自动记录部署历史
   - 支持快速回滚到任意历史版本
   - 保留可配置数量的历史版本

4. **灵活的更新策略**
   - 支持滚动更新（RollingUpdate）
   - 支持重建更新（Recreate）
   - 可自定义更新速度和并发度

5. **扩缩容能力**
   - 支持手动扩缩容
   - 可与 HPA（Horizontal Pod Autoscaler）集成
   - 支持按比例缩放

### 🔗 与 ReplicaSet 的关系

Deployment 和 ReplicaSet 的关系可以用"管理者"和"执行者"来形容：

```
┌─────────────────────────────────────────────────┐
│                   Deployment                     │
│  ┌───────────────────────────────────────────┐  │
│  │ 版本管理 │ 更新策略 │ 回滚机制 │ 声明式API │  │
│  └───────────────────────────────────────────┘  │
│                       ↓                          │
│  ┌──────────────┐  ┌──────────────┐            │
│  │ ReplicaSet   │  │ ReplicaSet   │            │
│  │ (version 1)  │  │ (version 2)  │            │
│  └──────────────┘  └──────────────┘            │
│         ↓                   ↓                    │
│    ┌─────────┐         ┌─────────┐              │
│    │  Pods   │         │  Pods   │              │
│    └─────────┘         └─────────┘              │
└─────────────────────────────────────────────────┘
```

**关键关系特征：**

1. **一对多关系**
   - 一个 Deployment 可以管理多个 ReplicaSet
   - 每个 ReplicaSet 对应一个特定版本的应用

2. **生命周期管理**
   - Deployment 创建时自动创建 ReplicaSet
   - 更新时创建新的 ReplicaSet
   - 根据保留策略清理旧的 ReplicaSet

3. **职责分工**
   - **Deployment**：负责版本管理、更新策略、回滚等高级功能
   - **ReplicaSet**：负责确保指定数量的 Pod 副本始终运行

4. **版本对应**
   ```bash
   # Check the relationship
   kubectl get deployment nginx-deployment -o wide
   kubectl get replicaset -l app=nginx
   kubectl get pods -l app=nginx
   ```

## Deployment 的核心概念

### 1. ReplicaSet 管理

#### 自动创建与管理

Deployment 会自动创建和管理 ReplicaSet，每次更新都会产生新的 ReplicaSet：

```yaml
# When you update the Deployment
spec:
  template:
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2  # Changed from 1.14.1
```

结果：
```bash
$ kubectl get rs
NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-74d6b68f8d   3         3         3       5m    # New RS
nginx-deployment-65f84cd6fc   0         0         0       10m   # Old RS
```

#### ReplicaSet 命名规则

- 格式：`[deployment-name]-[pod-template-hash]`
- Pod 模板哈希值确保唯一性
- 相同的 Pod 模板生成相同的哈希值

#### 版本控制

```bash
# View revision history
kubectl rollout history deployment/nginx-deployment

# Check specific revision
kubectl rollout history deployment/nginx-deployment --revision=2
```

### 2. 滚动更新机制

#### 工作原理

滚动更新是 Deployment 的默认更新策略，它通过逐步替换旧版本 Pod 来实现零停机更新：

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%        # Maximum pods above desired number
      maxUnavailable: 25%  # Maximum pods unavailable during update
```

#### 更新流程

1. **创建新 ReplicaSet**
   ```
   Initial: RS-v1 (3 pods running)
   Step 1:  RS-v1 (3 pods) + RS-v2 (0 pods) created
   ```

2. **逐步迁移**
   ```
   Step 2:  RS-v1 (2 pods) + RS-v2 (1 pod)
   Step 3:  RS-v1 (1 pod)  + RS-v2 (2 pods)
   Step 4:  RS-v1 (0 pods) + RS-v2 (3 pods)
   ```

3. **清理旧版本**
   ```
   Final:   RS-v2 (3 pods running)
            RS-v1 (0 pods, kept for rollback)
   ```

#### 更新参数详解

- **maxSurge**: 更新过程中可以创建的额外 Pod 数量
  - 可以是数字或百分比
  - 决定更新速度
  - 影响资源使用

- **maxUnavailable**: 更新过程中不可用的 Pod 最大数量
  - 可以是数字或百分比
  - 影响服务可用性
  - 不能和 maxSurge 同时为 0

#### 实际案例

```yaml
# Fast update with more resources
spec:
  replicas: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 5          # Can have up to 15 pods
      maxUnavailable: 0    # Always maintain 10 available pods

# Conservative update
spec:
  replicas: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1          # Max 11 pods
      maxUnavailable: 1    # Min 9 available pods
```

### 3. 版本历史和回滚

#### 历史记录管理

```yaml
spec:
  revisionHistoryLimit: 10  # Keep 10 old ReplicaSets
```

#### 查看历史

```bash
# List all revisions
kubectl rollout history deployment/nginx-deployment

# View specific revision details
kubectl rollout history deployment/nginx-deployment --revision=2

# Compare revisions
kubectl diff -f deployment.yaml
```

#### 回滚操作

```bash
# Rollback to previous version
kubectl rollout undo deployment/nginx-deployment

# Rollback to specific revision
kubectl rollout undo deployment/nginx-deployment --to-revision=2

# Check rollback status
kubectl rollout status deployment/nginx-deployment
```

#### 回滚原理

1. Deployment 控制器识别目标修订版本
2. 获取该版本对应的 ReplicaSet
3. 开始滚动更新到目标版本
4. 更新 Deployment 的 revision 注解

### 4. 更新策略（RollingUpdate、Recreate）

#### RollingUpdate（滚动更新）

**特点：**
- 逐步替换 Pod
- 保持服务可用
- 可控制更新速度

**适用场景：**
- 生产环境
- 需要零停机时间
- 有负载均衡器

**配置示例：**
```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
```

#### Recreate（重建）

**特点：**
- 先删除所有旧 Pod
- 再创建新 Pod
- 存在服务中断

**适用场景：**
- 开发/测试环境
- 资源受限
- 不支持多版本共存

**配置示例：**
```yaml
spec:
  strategy:
    type: Recreate
```

#### 策略对比

| 特性 | RollingUpdate | Recreate |
|------|---------------|----------|
| **停机时间** | 无 | 有 |
| **资源需求** | 需要额外资源 | 不需要额外资源 |
| **更新速度** | 可控 | 快速 |
| **版本共存** | 支持 | 不支持 |
| **复杂度** | 较高 | 简单 |
| **适用环境** | 生产 | 开发/测试 |

### 5. Pod 模板和选择器

#### Pod 模板

Pod 模板定义了 Deployment 创建的 Pod 规格：

```yaml
spec:
  template:
    metadata:
      labels:
        app: nginx
        version: v1
      annotations:
        prometheus.io/scrape: "true"
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
```

#### 选择器

选择器定义了 Deployment 管理哪些 Pod：

```yaml
spec:
  selector:
    matchLabels:
      app: nginx
  # Or use more complex selectors
  selector:
    matchExpressions:
    - key: app
      operator: In
      values:
      - nginx
      - nginx-canary
```

#### 重要规则

1. **不可变性**
   - 选择器一旦设置不能修改
   - 修改需要删除并重建 Deployment

2. **匹配要求**
   - Pod 模板的标签必须匹配选择器
   - 否则 API 会拒绝创建

3. **唯一性**
   - 选择器应该是唯一的
   - 避免多个 Deployment 管理同一组 Pod

#### 最佳实践

```yaml
# Good: Specific and unique labels
spec:
  selector:
    matchLabels:
      app: nginx
      component: frontend
      environment: production
  template:
    metadata:
      labels:
        app: nginx
        component: frontend
        environment: production
        version: v1.2.3  # Additional labels for monitoring

# Bad: Too generic
spec:
  selector:
    matchLabels:
      app: web  # Too generic, might conflict
```

### 🎯 核心概念总结

1. **ReplicaSet 管理**：Deployment 自动创建和管理 ReplicaSet，实现版本控制
2. **滚动更新**：通过 maxSurge 和 maxUnavailable 控制更新过程
3. **版本历史**：自动保留历史版本，支持快速回滚
4. **更新策略**：RollingUpdate 适合生产环境，Recreate 适合开发测试
5. **Pod 模板和选择器**：定义 Pod 规格和管理范围，确保精确控制

## Deployment 配置详解

### 📋 基本结构示例

以下是一个完整的 Deployment YAML 配置示例，包含了所有重要的配置字段：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-deployment
  namespace: production
  labels:
    app: webapp
    tier: frontend
  annotations:
    kubernetes.io/change-cause: "Update webapp to version 2.0"
spec:
  replicas: 5                      # Number of desired pods
  revisionHistoryLimit: 10         # Number of old ReplicaSets to retain
  progressDeadlineSeconds: 600     # Maximum time for deployment to make progress
  paused: false                    # Whether the deployment is paused
  
  selector:                        # Label selector for pods
    matchLabels:
      app: webapp
      tier: frontend
  
  strategy:                        # Update strategy
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2                  # Maximum number of pods above desired replicas
      maxUnavailable: 1            # Maximum number of unavailable pods
  
  template:                        # Pod template
    metadata:
      labels:
        app: webapp
        tier: frontend
        version: v2.0
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9090"
    spec:
      containers:
      - name: webapp
        image: myregistry/webapp:2.0
        imagePullPolicy: IfNotPresent
        ports:
        - name: http
          containerPort: 8080
          protocol: TCP
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
        livenessProbe:
          httpGet:
            path: /health
            port: http
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /ready
            port: http
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 3
          successThreshold: 1
        env:
        - name: LOG_LEVEL
          value: "info"
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: url
      restartPolicy: Always
      terminationGracePeriodSeconds: 30
```

### 🔍 关键字段详解

#### 1. spec.replicas

**定义**：期望运行的 Pod 副本数量

**类型**：整数（默认值：1）

**用途**：
- 定义应用的规模
- 实现负载均衡
- 提供高可用性

**示例**：
```yaml
spec:
  replicas: 3  # Maintain 3 pods at all times
```

**动态调整**：
```bash
# Scale up/down manually
kubectl scale deployment webapp-deployment --replicas=10

# Auto-scaling with HPA
kubectl autoscale deployment webapp-deployment --min=3 --max=10 --cpu-percent=80
```

**注意事项**：
- 设置为 0 会停止所有 Pod
- 与 HPA 一起使用时，HPA 会覆盖此值
- 更改此值会触发扩缩容操作

#### 2. spec.selector

**定义**：用于选择由此 Deployment 管理的 Pod 的标签选择器

**类型**：LabelSelector

**用途**：
- 确定哪些 Pod 属于此 Deployment
- 与 Pod 模板的标签建立关联
- 实现精确的 Pod 管理

**示例**：
```yaml
# Simple label selector
spec:
  selector:
    matchLabels:
      app: webapp
      tier: frontend

# Advanced selector with expressions
spec:
  selector:
    matchExpressions:
    - key: app
      operator: In
      values: [webapp, web-app]
    - key: tier
      operator: NotIn
      values: [backend]
    - key: environment
      operator: Exists
```

**操作符说明**：
- `In`：标签值在指定列表中
- `NotIn`：标签值不在指定列表中
- `Exists`：标签键存在
- `DoesNotExist`：标签键不存在

**重要规则**：
- 一旦设置，选择器不可更改
- 必须与 Pod 模板的标签匹配
- 应该确保唯一性，避免选择到其他 Deployment 的 Pod

#### 3. spec.template

**定义**：创建新 Pod 时使用的模板

**类型**：PodTemplateSpec

**用途**：
- 定义 Pod 的完整规格
- 包含容器配置、存储、网络等
- 更新此部分会触发滚动更新

**结构**：
```yaml
spec:
  template:
    metadata:           # Pod metadata
      labels:          # Required: must match selector
        app: webapp
      annotations:     # Optional: additional metadata
        version: "2.0"
    spec:              # Pod specification
      containers:      # Container definitions
      volumes:         # Volume definitions
      initContainers:  # Init containers
      nodeSelector:    # Node selection constraints
      affinity:        # Advanced scheduling
      tolerations:     # Toleration rules
      serviceAccountName: # Service account
```

**更新触发条件**：
- 容器镜像变更
- 环境变量修改
- 资源限制调整
- 任何 Pod spec 的改变

#### 4. spec.strategy（重点）

**定义**：部署更新策略，控制如何将现有 Pod 替换为新 Pod

**类型**：DeploymentStrategy

**策略类型**：

##### RollingUpdate（默认）

**配置参数**：
```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%        # Can be number or percentage
      maxUnavailable: 25%  # Can be number or percentage
```

**maxSurge 详解**：
- **定义**：更新期间可以创建的超出期望副本数的 Pod 数量
- **作用**：控制更新速度和资源使用
- **示例场景**：
  ```yaml
  # Example 1: Percentage
  replicas: 10
  maxSurge: 30%  # Can have up to 13 pods (10 + 3)
  
  # Example 2: Absolute number
  replicas: 10
  maxSurge: 5    # Can have up to 15 pods (10 + 5)
  ```

**maxUnavailable 详解**：
- **定义**：更新期间不可用的 Pod 最大数量
- **作用**：保证服务可用性
- **示例场景**：
  ```yaml
  # Example 1: Zero downtime
  replicas: 10
  maxUnavailable: 0  # Always keep 10 pods running
  
  # Example 2: Allow some downtime
  replicas: 10
  maxUnavailable: 2  # Minimum 8 pods available
  ```

**更新流程示例**：
```yaml
# Initial state: 5 replicas
spec:
  replicas: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2         # Max 7 pods total
      maxUnavailable: 1   # Min 4 pods available

# Update process:
# Step 1: Scale up new RS to 2 (total: 7 pods)
# Step 2: Scale down old RS by 1 (total: 6 pods, 4 old + 2 new)
# Step 3: Continue until all pods are updated
```

##### Recreate 策略

```yaml
spec:
  strategy:
    type: Recreate
# No additional parameters needed
```

**特点**：
- 先终止所有现有 Pod
- 然后创建新 Pod
- 导致服务中断
- 适用于不能多版本共存的应用

**策略选择指南**：

| 场景 | 推荐策略 | 配置建议 |
|------|---------|----------|
| **生产环境-高可用要求** | RollingUpdate | maxUnavailable: 0, maxSurge: 1-2 |
| **生产环境-资源受限** | RollingUpdate | maxUnavailable: 1, maxSurge: 1 |
| **快速更新** | RollingUpdate | maxUnavailable: 50%, maxSurge: 50% |
| **数据库迁移** | Recreate | - |
| **开发环境** | Recreate 或 RollingUpdate | 根据需求选择 |

#### 5. spec.revisionHistoryLimit

**定义**：保留的历史 ReplicaSet 数量

**类型**：整数（默认值：10）

**用途**：
- 支持回滚操作
- 保留部署历史
- 控制资源使用

**示例**：
```yaml
spec:
  revisionHistoryLimit: 5  # Keep last 5 revisions
```

**管理命令**：
```bash
# View revision history
kubectl rollout history deployment webapp-deployment

# Clean up old ReplicaSets manually
kubectl delete rs -l app=webapp --field-selector status.replicas=0
```

**最佳实践**：
- 生产环境：10-20（便于回滚）
- 开发环境：3-5（节省资源）
- 频繁更新的应用：可适当增加

#### 6. spec.progressDeadlineSeconds

**定义**：Deployment 进展的最大秒数，超时则认为失败

**类型**：整数（默认值：600 秒）

**用途**：
- 防止部署无限期挂起
- 及时发现部署问题
- 触发自动回滚（如配置）

**示例**：
```yaml
spec:
  progressDeadlineSeconds: 300  # 5 minutes timeout
```

**进展条件**：
- 创建新 Pod
- 删除旧 Pod
- Pod 变为 Ready 状态

**超时后果**：
- Deployment 状态显示为失败
- 停止继续更新
- 可触发告警或自动回滚

**查看状态**：
```bash
# Check deployment conditions
kubectl describe deployment webapp-deployment | grep -A5 Conditions

# Example output when timeout
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Progressing    False   ProgressDeadlineExceeded
  Available      True    MinimumReplicasAvailable
```

#### 7. spec.paused

**定义**：是否暂停 Deployment 的更新

**类型**：布尔值（默认值：false）

**用途**：
- 金丝雀发布
- 分阶段部署
- 调试和测试

**示例**：
```yaml
spec:
  paused: true  # Deployment is paused
```

**使用场景**：

1. **金丝雀发布**：
   ```bash
   # Update image and pause
   kubectl set image deployment/webapp webapp=webapp:2.0
   kubectl rollout pause deployment/webapp
   
   # Check metrics, logs, etc.
   # If everything is OK, resume
   kubectl rollout resume deployment/webapp
   ```

2. **批量更新**：
   ```bash
   # Pause deployment
   kubectl rollout pause deployment/webapp
   
   # Make multiple changes
   kubectl set image deployment/webapp webapp=webapp:2.0
   kubectl set resources deployment/webapp -c=webapp --limits=memory=512Mi
   kubectl set env deployment/webapp LOG_LEVEL=debug
   
   # Resume to apply all changes at once
   kubectl rollout resume deployment/webapp
   ```

3. **紧急响应**：
   ```bash
   # Pause problematic deployment immediately
   kubectl rollout pause deployment/webapp
   
   # Investigate and fix issues
   # Resume when ready
   kubectl rollout resume deployment/webapp
   ```

### 📊 配置最佳实践

#### 1. 生产环境配置模板

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: production-app
  namespace: production
spec:
  replicas: 10
  revisionHistoryLimit: 10
  progressDeadlineSeconds: 600
  selector:
    matchLabels:
      app: production-app
      env: production
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2         # Conservative update
      maxUnavailable: 0   # Zero downtime
  template:
    metadata:
      labels:
        app: production-app
        env: production
    spec:
      containers:
      - name: app
        image: myapp:stable
        resources:
          requests:
            memory: "256Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "200m"
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

#### 2. 开发环境配置模板

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dev-app
  namespace: development
spec:
  replicas: 2
  revisionHistoryLimit: 3
  progressDeadlineSeconds: 300
  selector:
    matchLabels:
      app: dev-app
  strategy:
    type: Recreate       # Fast update, downtime acceptable
  template:
    metadata:
      labels:
        app: dev-app
    spec:
      containers:
      - name: app
        image: myapp:latest
        imagePullPolicy: Always  # Always pull latest
```

#### 3. 配置检查清单

- [ ] replicas 数量是否合适？
- [ ] selector 标签是否与 template 匹配？
- [ ] strategy 是否适合应用特性？
- [ ] revisionHistoryLimit 是否满足回滚需求？
- [ ] progressDeadlineSeconds 是否给予足够时间？
- [ ] 资源限制是否设置？
- [ ] 健康检查是否配置？
- [ ] 是否需要设置 node affinity？
- [ ] 是否需要配置 PodDisruptionBudget？

### 🎯 配置详解总结

1. **spec.replicas**：控制 Pod 数量，实现负载均衡和高可用
2. **spec.selector**：精确选择要管理的 Pod，一旦设置不可更改
3. **spec.template**：定义 Pod 规格，任何修改都会触发更新
4. **spec.strategy**：核心配置，决定更新方式和速度
5. **spec.revisionHistoryLimit**：控制历史版本保留，支持回滚
6. **spec.progressDeadlineSeconds**：设置超时保护，防止部署挂起
7. **spec.paused**：支持暂停功能，实现金丝雀发布等高级部署模式
