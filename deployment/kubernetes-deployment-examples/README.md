# Kubernetes Deployment 实战示例

本目录包含了5个使用 `richardchesterwood/k8s-fleetman-webapp-angular:release0` 镜像的 Kubernetes Deployment 示例，从基础到高级配置。

## 示例列表

### 1. 基础 Deployment (1-basic-deployment.yaml)
- **用途**: 最简单的 Deployment 配置
- **特点**:
  - 3个副本
  - 基本的容器端口配置
  - ClusterIP 类型的 Service
- **适用场景**: 快速部署应用，内部服务访问

### 2. 滚动更新示例 (2-rolling-update-deployment.yaml)
- **用途**: 展示滚动更新策略的详细配置
- **特点**:
  - 10个副本
  - maxSurge: 2 (最多可以多创建2个Pod)
  - maxUnavailable: 1 (更新时最多1个Pod不可用)
  - 配置了健康检查探针
  - LoadBalancer 类型的 Service
- **适用场景**: 需要零停机更新的生产环境应用

### 3. Recreate 策略示例 (3-recreate-deployment.yaml)
- **用途**: 展示 Recreate 更新策略
- **特点**:
  - 5个副本
  - Recreate 策略 (先停止所有旧Pod，再创建新Pod)
  - 生命周期钩子配置
  - 环境变量配置
  - 数据卷挂载
- **适用场景**: 数据库等不支持多版本同时运行的应用

### 4. 多容器 Pod Deployment (4-multi-container-deployment.yaml)
- **用途**: 展示多容器Pod的配置
- **包含容器**:
  - 主应用容器 (webapp)
  - 日志聚合 sidecar (log-aggregator)
  - 指标导出 sidecar (metrics-exporter)
  - 初始化容器 (config-init)
- **特点**:
  - 容器间共享卷
  - Init Container 预配置
  - 多端口 Service
- **适用场景**: 需要辅助功能的微服务架构

### 5. 高级配置示例 (5-advanced-deployment.yaml)
- **用途**: 展示企业级生产环境的完整配置
- **包含功能**:
  - 完整的健康检查 (liveness, readiness, startup)
  - 资源限制和请求
  - 安全上下文配置
  - Pod反亲和性规则
  - 节点选择器和容忍度
  - HPA (水平自动扩缩容)
  - PDB (Pod中断预算)
  - 多种环境变量源
  - ConfigMap 和 Secret 挂载
  - DNS 配置
  - 服务账户
- **适用场景**: 生产环境的关键业务应用

## 使用方法

1. **部署基础示例**:
```bash
kubectl apply -f 1-basic-deployment.yaml
```

2. **查看部署状态**:
```bash
kubectl get deployments
kubectl get pods -l app=fleetman-webapp
kubectl get services
```

3. **测试滚动更新**:
```bash
# 部署初始版本
kubectl apply -f 2-rolling-update-deployment.yaml

# 修改镜像版本后重新应用
kubectl set image deployment/fleetman-webapp-rolling webapp=richardchesterwood/k8s-fleetman-webapp-angular:release0-v2

# 查看更新过程
kubectl rollout status deployment/fleetman-webapp-rolling
```

4. **测试 Recreate 策略**:
```bash
kubectl apply -f 3-recreate-deployment.yaml
# 修改配置后重新应用，观察所有Pod同时终止和重建
```

5. **部署多容器示例**:
```bash
kubectl apply -f 4-multi-container-deployment.yaml
# 查看多个容器的日志
kubectl logs <pod-name> -c webapp
kubectl logs <pod-name> -c log-aggregator
```

6. **部署高级示例**:
```bash
# 先创建必要的资源（如果需要）
kubectl create secret generic webapp-advanced-secrets --from-literal=api-key=your-key
kubectl create configmap webapp-config --from-literal=feature-flag=true

# 部署应用
kubectl apply -f 5-advanced-deployment.yaml

# 查看HPA状态
kubectl get hpa

# 查看PDB状态
kubectl get pdb
```

## 清理资源

删除所有创建的资源：
```bash
kubectl delete -f .
```

## 注意事项

1. **镜像拉取**: 确保集群能够访问 Docker Hub 拉取镜像
2. **资源配额**: 高级示例需要较多资源，确保集群有足够的资源
3. **LoadBalancer**: 在某些环境（如 Minikube）中，LoadBalancer 类型的 Service 可能需要额外配置
4. **节点标签**: 高级示例使用了节点选择器，可能需要给节点打标签：
   ```bash
   kubectl label nodes <node-name> disktype=ssd environment=production
   ```

## 最佳实践

1. **始终设置资源限制**: 防止容器消耗过多资源
2. **配置健康检查**: 确保应用的可用性
3. **使用标签**: 便于管理和选择资源
4. **设置PDB**: 在集群维护时保证服务可用性
5. **使用HPA**: 根据负载自动调整副本数
6. **安全配置**: 使用安全上下文限制容器权限
