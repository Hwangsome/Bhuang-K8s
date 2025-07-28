# Kubernetes Deployment 操作指南

本指南详细介绍 Kubernetes Deployment 的各种常用操作命令，包括创建、查看、更新、回滚、扩缩容等操作。

## 1. 创建和查看 Deployment

### 1.1 创建 Deployment

#### 使用 YAML 文件创建
```bash
# Create deployment from YAML file
kubectl create -f deployment.yaml

# Or use apply (recommended for declarative management)
kubectl apply -f deployment.yaml
```

#### 使用命令行创建
```bash
# Create a deployment with a single command
kubectl create deployment nginx-deployment --image=nginx:1.14.2

# Create deployment with replicas
kubectl create deployment nginx-deployment --image=nginx:1.14.2 --replicas=3

# Dry run to generate YAML
kubectl create deployment nginx-deployment --image=nginx:1.14.2 --dry-run=client -o yaml > deployment.yaml
```

### 1.2 查看 Deployment

#### 列出所有 Deployment
```bash
# List all deployments in current namespace
kubectl get deployments

# List deployments with more details
kubectl get deployments -o wide

# List deployments in all namespaces
kubectl get deployments --all-namespaces

# Short alias
kubectl get deploy
```

#### 查看特定 Deployment 详情
```bash
# Get detailed information about a deployment
kubectl describe deployment nginx-deployment

# Get deployment in YAML format
kubectl get deployment nginx-deployment -o yaml

# Get deployment in JSON format
kubectl get deployment nginx-deployment -o json
```

#### 查看 Deployment 状态
```bash
# Watch deployment status in real-time
kubectl get deployment nginx-deployment --watch

# Get deployment status
kubectl rollout status deployment/nginx-deployment
```

## 2. 更新操作

### 2.1 使用 set image 更新镜像

```bash
# Update container image
kubectl set image deployment/nginx-deployment nginx=nginx:1.16.1

# Update multiple containers
kubectl set image deployment/nginx-deployment nginx=nginx:1.16.1 sidecar=sidecar:v2

# Update all deployments with specific image
kubectl set image deployments --all nginx=nginx:1.16.1

# Record the change for rollback history
kubectl set image deployment/nginx-deployment nginx=nginx:1.16.1 --record
```

### 2.2 使用 edit 直接编辑

```bash
# Edit deployment in default editor
kubectl edit deployment nginx-deployment

# Use specific editor
KUBE_EDITOR="vim" kubectl edit deployment nginx-deployment
```

### 2.3 使用 apply 更新

```bash
# Update deployment by applying modified YAML
kubectl apply -f deployment.yaml

# Apply with record flag for history
kubectl apply -f deployment.yaml --record

# Server-side apply (recommended for GitOps)
kubectl apply -f deployment.yaml --server-side
```

### 2.4 更新其他配置

```bash
# Update replicas
kubectl scale deployment nginx-deployment --replicas=5

# Update environment variables
kubectl set env deployment/nginx-deployment ENV_VAR=value

# Update resources
kubectl set resources deployment nginx-deployment -c=nginx --limits=cpu=200m,memory=512Mi

# Update service account
kubectl set serviceaccount deployment nginx-deployment new-service-account
```

## 3. 回滚操作

### 3.1 查看更新历史

```bash
# View rollout history
kubectl rollout history deployment/nginx-deployment

# View specific revision details
kubectl rollout history deployment/nginx-deployment --revision=2
```

### 3.2 回滚到上一个版本

```bash
# Rollback to previous version
kubectl rollout undo deployment/nginx-deployment

# Check rollback status
kubectl rollout status deployment/nginx-deployment
```

### 3.3 回滚到指定版本

```bash
# Rollback to specific revision
kubectl rollout undo deployment/nginx-deployment --to-revision=2

# Dry run to see what would change
kubectl rollout undo deployment/nginx-deployment --to-revision=2 --dry-run=client
```

## 4. 扩缩容操作

### 4.1 手动扩缩容

```bash
# Scale up to 5 replicas
kubectl scale deployment nginx-deployment --replicas=5

# Scale down to 2 replicas
kubectl scale deployment nginx-deployment --replicas=2

# Scale multiple deployments
kubectl scale deployment nginx-deployment mysql-deployment --replicas=3

# Scale if current replicas match condition
kubectl scale deployment nginx-deployment --current-replicas=2 --replicas=3
```

### 4.2 自动扩缩容 (HPA)

```bash
# Create Horizontal Pod Autoscaler
kubectl autoscale deployment nginx-deployment --cpu-percent=50 --min=1 --max=10

# View HPA status
kubectl get hpa

# Describe HPA details
kubectl describe hpa nginx-deployment-hpa

# Delete HPA
kubectl delete hpa nginx-deployment-hpa
```

### 4.3 基于自定义指标的扩缩容

```yaml
# Create HPA with custom metrics (save as hpa-custom.yaml)
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: nginx-deployment-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx-deployment
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

```bash
# Apply custom HPA
kubectl apply -f hpa-custom.yaml
```

## 5. 暂停和恢复更新

### 5.1 暂停 Deployment 更新

```bash
# Pause deployment rollout
kubectl rollout pause deployment/nginx-deployment

# Make multiple changes while paused
kubectl set image deployment/nginx-deployment nginx=nginx:1.16.1
kubectl set resources deployment nginx-deployment -c=nginx --limits=cpu=200m,memory=512Mi
kubectl scale deployment nginx-deployment --replicas=5
```

### 5.2 恢复 Deployment 更新

```bash
# Resume deployment rollout
kubectl rollout resume deployment/nginx-deployment

# Check rollout status after resume
kubectl rollout status deployment/nginx-deployment
```

## 6. 查看更新历史和状态

### 6.1 查看 Deployment 事件

```bash
# View deployment events
kubectl describe deployment nginx-deployment | grep -A 10 Events

# View all events related to deployment
kubectl get events --field-selector involvedObject.name=nginx-deployment
```

### 6.2 查看 ReplicaSet 历史

```bash
# List ReplicaSets owned by deployment
kubectl get rs -l app=nginx

# View old ReplicaSets
kubectl get rs -l app=nginx --show-labels
```

### 6.3 查看 Pod 状态

```bash
# View pods created by deployment
kubectl get pods -l app=nginx

# Watch pod status changes
kubectl get pods -l app=nginx --watch

# View pod logs
kubectl logs -l app=nginx --tail=100

# View previous container logs
kubectl logs -l app=nginx --previous
```

### 6.4 监控 Deployment 指标

```bash
# View resource usage
kubectl top pods -l app=nginx

# View deployment metrics (requires metrics-server)
kubectl get deployment nginx-deployment --show-labels
kubectl describe deployment nginx-deployment | grep -E "Replicas:|Updated|Available"
```

## 7. 高级操作技巧

### 7.1 批量操作

```bash
# Update all deployments in namespace
kubectl set image deployment --all nginx=nginx:1.16.1

# Scale all deployments
kubectl scale deployment --all --replicas=3

# Delete all deployments with specific label
kubectl delete deployment -l environment=test
```

### 7.2 使用标签选择器

```bash
# Get deployments with specific label
kubectl get deployment -l app=nginx

# Update deployments with label selector
kubectl set image deployment -l version=v1 nginx=nginx:1.16.1
```

### 7.3 导出和备份

```bash
# Export deployment without cluster-specific information
kubectl get deployment nginx-deployment -o yaml --export > deployment-backup.yaml

# Export all deployments
kubectl get deployment --all-namespaces -o yaml > all-deployments-backup.yaml
```

### 7.4 调试技巧

```bash
# Get deployment with extended information
kubectl get deployment nginx-deployment -o wide

# View deployment conditions
kubectl get deployment nginx-deployment -o json | jq '.status.conditions'

# Check deployment strategy
kubectl get deployment nginx-deployment -o json | jq '.spec.strategy'

# View deployment annotations
kubectl get deployment nginx-deployment -o json | jq '.metadata.annotations'
```

## 8. 最佳实践建议

1. **始终使用 --record 标志**：记录变更历史，便于回滚
   ```bash
   kubectl apply -f deployment.yaml --record
   ```

2. **使用声明式管理**：优先使用 `kubectl apply` 而不是 `kubectl create`

3. **设置合适的更新策略**：
   - RollingUpdate：逐步更新，保证服务可用性
   - Recreate：删除所有旧 Pod 后创建新 Pod

4. **配置健康检查**：设置 readinessProbe 和 livenessProbe

5. **设置资源限制**：为容器设置 requests 和 limits

6. **使用命名空间隔离**：不同环境使用不同命名空间

7. **版本控制**：将 Deployment YAML 文件纳入版本控制系统

8. **监控和告警**：配置适当的监控和告警规则

## 9. 常见问题排查

### 9.1 Deployment 无法更新
```bash
# Check deployment status
kubectl rollout status deployment/nginx-deployment

# Check events
kubectl describe deployment nginx-deployment

# Check ReplicaSet
kubectl get rs -l app=nginx
```

### 9.2 Pod 无法启动
```bash
# Check pod status
kubectl get pods -l app=nginx

# View pod logs
kubectl logs <pod-name>

# Describe pod for events
kubectl describe pod <pod-name>
```

### 9.3 回滚失败
```bash
# Check rollout history
kubectl rollout history deployment/nginx-deployment

# Force rollback
kubectl rollout undo deployment/nginx-deployment --to-revision=1
```

这份指南涵盖了 Kubernetes Deployment 的所有常用操作，从基础的创建和查看，到高级的更新策略和故障排查。根据实际需求选择合适的命令和操作方式。
