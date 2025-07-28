# Kubernetes Service 示例说明

## 部署概览

我们创建了以下资源：
1. **Pod**: webapp
2. **Service**: webapp-service

## Pod 配置详解

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: webapp
  labels:
    app: webapp        # 这个标签很重要，Service 会通过它来选择 Pod
    type: frontend
spec:
  containers:
  - name: webapp
    image: richardchesterwood/k8s-fleetman-webapp-angular:release0
    ports:
    - containerPort: 80   # 容器暴露的端口
      name: http
```

## Service 配置详解

```yaml
apiVersion: v1
kind: Service
metadata:
  name: webapp-service
spec:
  selector:
    app: webapp          # 通过这个选择器找到标签为 app=webapp 的 Pod
  ports:
  - name: http
    port: 80            # Service 暴露的端口
    targetPort: 80      # 转发到 Pod 的端口
    nodePort: 30080     # Node 上暴露的端口（外部访问）
  type: NodePort        # Service 类型
```

## Service 工作流程

1. **Pod 选择**: Service 通过 `selector` 找到所有标签为 `app: webapp` 的 Pod
2. **Endpoints 创建**: Kubernetes 自动创建 Endpoints 对象，记录所有匹配的 Pod IP
3. **流量转发**: 
   - 集群内部：通过 ClusterIP (10.97.100.173:80) 访问
   - 集群外部：通过 NodePort (节点IP:30080) 访问

## 访问方式

### 1. 从集群内部访问
```bash
# 使用 Service 名称（在同一命名空间内）
curl http://webapp-service

# 使用 ClusterIP
curl http://10.97.100.173
```

### 2. 从集群外部访问
```bash
# 使用 NodePort
curl http://localhost:30080
```

### 3. 测试连接
```bash
# 创建一个临时 Pod 来测试内部访问
kubectl run test-pod --image=busybox -it --rm -- wget -qO- http://webapp-service

# 查看 Service 的 Endpoints
kubectl describe service webapp-service

# 查看 Endpoints 对象
kubectl get endpoints webapp-service
```

## Service 核心概念演示

### Selector 和 Labels 的关系
- Pod 有标签：`app: webapp`
- Service 有选择器：`selector: app: webapp`
- Service 会自动发现并路由到所有匹配的 Pod

### ClusterIP
- 自动分配的虚拟 IP：10.97.100.173
- 只能在集群内部访问
- 稳定不变，即使 Pod 重启

### NodePort
- 在每个节点上开放端口 30080
- 允许外部流量访问
- 端口范围：30000-32767

### Endpoints
- 动态维护 Pod IP 列表
- Pod IP：10.1.0.197
- 当 Pod 重启时自动更新

## 流量路径图解

```
外部客户端
    |
    v
NodePort (30080)
    |
    v
kube-proxy (iptables/IPVS)
    |
    v
Service ClusterIP (10.97.100.173:80)
    |
    v
Endpoints (10.1.0.197:80)
    |
    v
Pod webapp
```

这个示例完整展示了 Service 如何将外部流量路由到 Pod，体现了 Service 的核心功能。
