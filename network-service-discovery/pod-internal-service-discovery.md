# Pod 内容器间服务发现

## 概述

在 Kubernetes 中，同一个 Pod 内的多个容器共享网络命名空间，这意味着：
- 它们共享同一个 IP 地址
- 它们可以通过 `localhost` (127.0.0.1) 互相通信
- 它们共享端口空间（不能有端口冲突）

## 核心概念

### 网络命名空间共享

Pod 是 Kubernetes 中的最小部署单元，Pod 内的所有容器：
- 共享网络接口和 IP 地址
- 共享存储卷
- 作为一个整体被调度和管理

### 通信机制

```
┌─────────────────────────────────────┐
│             Pod                     │
│  ┌─────────────┐  ┌─────────────┐  │
│  │ Container A │  │ Container B │  │
│  │   Port 8080 │  │   Port 3000 │  │
│  └──────┬──────┘  └──────┬──────┘  │
│         │                 │         │
│         └────localhost────┘         │
│                                     │
│         Shared Network NS           │
│         IP: 10.244.1.5             │
└─────────────────────────────────────┘
```

## 使用场景

### 1. Sidecar 模式
- 日志收集器（如 Fluentd）从应用容器收集日志
- 服务网格代理（如 Envoy）处理进出流量

### 2. 辅助服务
- 本地缓存服务（如 Redis）
- 配置热更新服务
- 监控指标收集

### 3. 适配器模式
- 协议转换
- 数据格式转换
- API 适配

## 优势

1. **性能高效**：本地通信，无网络开销
2. **配置简单**：无需服务发现机制
3. **紧密耦合**：适合紧密协作的组件
4. **原子性**：容器作为整体调度和管理

## 限制

1. **扩展性**：不能独立扩展单个容器
2. **资源共享**：共享 CPU、内存限制
3. **生命周期**：容器故障可能影响整个 Pod
4. **端口冲突**：需要协调端口使用

## 最佳实践

### 1. 端口规划
```yaml
# 为每个容器分配明确的端口范围
- name: web-app
  ports:
  - containerPort: 8080  # 应用端口
- name: sidecar
  ports:
  - containerPort: 9090  # 监控端口
```

### 2. 健康检查
```yaml
# 每个容器独立的健康检查
livenessProbe:
  httpGet:
    path: /health
    port: 8080
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
```

### 3. 资源限制
```yaml
# 合理分配资源，避免争抢
resources:
  requests:
    memory: "64Mi"
    cpu: "250m"
  limits:
    memory: "128Mi"
    cpu: "500m"
```

### 4. 启动顺序
使用 init containers 或启动探针确保依赖服务就绪：
```yaml
initContainers:
- name: wait-for-db
  image: busybox
  command: ['sh', '-c', 'until nc -z localhost 3306; do sleep 1; done']
```

## 示例应用架构

### Web 应用 + 本地缓存
```
Pod: web-service
├── nginx (port 80) - 反向代理
├── app (port 8080) - 业务应用
└── redis (port 6379) - 本地缓存
```

### 微服务 + Sidecar
```
Pod: microservice
├── service (port 8080) - 微服务
├── envoy (port 15000) - 服务网格代理
└── fluentd (port 24224) - 日志收集
```

## 调试技巧

### 1. 进入容器测试连接
```bash
# 进入容器 A
kubectl exec -it <pod-name> -c <container-a> -- /bin/sh

# 测试连接容器 B
curl localhost:3000
wget -O- localhost:3000
nc -zv localhost 3000
```

### 2. 查看网络配置
```bash
# 查看 Pod 的网络接口
kubectl exec <pod-name> -c <container> -- ip addr

# 查看监听端口
kubectl exec <pod-name> -c <container> -- netstat -tlnp
```

### 3. 查看 Pod 描述
```bash
# 查看 Pod 详细信息，包括所有容器
kubectl describe pod <pod-name>
```

## 注意事项

1. **不要**用于松耦合的服务 - 使用 Service
2. **不要**用于需要独立扩展的组件
3. **确保**容器间的启动依赖关系
4. **避免**端口冲突
5. **考虑**失败隔离 - 一个容器失败可能影响整个 Pod

## 总结

Pod 内容器间通过 localhost 通信是 Kubernetes 提供的基础网络功能，适合紧密耦合的组件。理解这一机制对于设计 Sidecar 模式、本地辅助服务等架构模式至关重要。在实际应用中，需要权衡其便利性和限制，选择合适的架构模式。
