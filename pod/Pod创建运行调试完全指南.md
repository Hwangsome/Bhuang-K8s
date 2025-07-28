# Kubernetes Pod 创建、运行与调试完全指南

## 目录
1. [Pod 基础概念](#pod-基础概念)
2. [创建 Pod](#创建-pod)
3. [运行和管理 Pod](#运行和管理-pod)
4. [访问 Pod](#访问-pod)
5. [调试 Pod](#调试-pod)
6. [常见问题与解决方案](#常见问题与解决方案)
7. [最佳实践](#最佳实践)

## Pod 基础概念

Pod 是 Kubernetes 中最小的可部署单元，包含一个或多个容器。

### Pod 的关键组成部分：
- **apiVersion**: API 版本（通常是 v1）
- **kind**: 资源类型（Pod）
- **metadata**: 元数据（名称、标签等）
- **spec**: 规格定义（容器、存储、网络等）

## 创建 Pod

### 1. 编写 Pod YAML 文件

#### 基础 Pod 示例
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  labels:
    app: myapp
spec:
  containers:
  - name: my-container
    image: nginx:latest
    ports:
    - containerPort: 80
```

#### 完整 Pod 示例（包含所有常用配置）
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: webapp-pod
  labels:
    app: webapp
    tier: frontend
    version: v1
spec:
  containers:
  - name: webapp
    image: myapp:latest
    ports:
    - containerPort: 80
      name: http
      protocol: TCP
    # 资源限制
    resources:
      limits:
        memory: "256Mi"
        cpu: "500m"
      requests:
        memory: "128Mi"
        cpu: "250m"
    # 环境变量
    env:
    - name: ENV_VAR_1
      value: "value1"
    - name: ENV_VAR_2
      value: "value2"
    # 健康检查
    livenessProbe:
      httpGet:
        path: /health
        port: 80
      initialDelaySeconds: 30
      periodSeconds: 10
    readinessProbe:
      httpGet:
        path: /ready
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 5
    # 挂载卷
    volumeMounts:
    - name: config
      mountPath: /etc/config
    - name: data
      mountPath: /data
  # 重启策略
  restartPolicy: Always
  # 卷定义
  volumes:
  - name: config
    configMap:
      name: app-config
  - name: data
    emptyDir: {}
```

### 2. 验证 YAML 文件

```bash
# 检查 YAML 语法（需要安装 yamllint）
yamllint pod.yaml

# 使用 kubectl 验证
kubectl apply --dry-run=client -f pod.yaml

# 检查是否使用了制表符（应该使用空格）
grep -P '\t' pod.yaml
```

## 运行和管理 Pod

### 1. 创建 Pod
```bash
# 从文件创建
kubectl apply -f pod.yaml

# 或使用 create（不可重复执行）
kubectl create -f pod.yaml

# 使用命令行快速创建
kubectl run nginx-pod --image=nginx:latest --port=80
```

### 2. 查看 Pod 状态
```bash
# 基本状态
kubectl get pods

# 详细信息
kubectl get pods -o wide

# 特定 Pod
kubectl get pod my-pod

# 实时监控
kubectl get pods -w

# YAML 格式输出
kubectl get pod my-pod -o yaml
```

### 3. 查看 Pod 详情
```bash
# 详细描述
kubectl describe pod my-pod

# 查看事件
kubectl get events --field-selector involvedObject.name=my-pod
```

### 4. 管理 Pod
```bash
# 删除 Pod
kubectl delete pod my-pod

# 强制删除
kubectl delete pod my-pod --force --grace-period=0

# 编辑运行中的 Pod（有限制）
kubectl edit pod my-pod
```

## 访问 Pod

### 重要说明：Pod 网络隔离
**Pod 默认只能在 Kubernetes 集群内部网络访问！**
- Pod IP（如 10.1.x.x）是集群内部 IP，外部无法直接访问
- 必须通过 Service、port-forward 或其他方式暴露

### Docker Desktop 特别说明（Mac/Windows）
在 Docker Desktop 上：
- K8s 运行在虚拟机内，Pod IP 在 VM 内部
- NodePort 和 LoadBalancer 自动映射到 localhost
- 不能直接访问 Pod IP 和 ClusterIP

### 1. 端口转发（最简单）
```bash
# 转发本地端口到 Pod
kubectl port-forward my-pod 8080:80

# 指定地址
kubectl port-forward --address 0.0.0.0 my-pod 8080:80

# 后台运行
kubectl port-forward my-pod 8080:80 &

# Docker Desktop 访问
open http://localhost:8080  # Mac
start http://localhost:8080 # Windows
```

### 2. 进入 Pod 容器

#### 重要提示：不是所有容器都有 bash！
很多容器（特别是基于 Alpine Linux）没有 bash，只有 sh。

```bash
# 推荐：先尝试 sh（最通用）
kubectl exec -it my-pod -- /bin/sh

# 如果容器有 bash
kubectl exec -it my-pod -- /bin/bash

# 执行单个命令
kubectl exec my-pod -- ls /app

# 多容器 Pod 指定容器
kubectl exec -it my-pod -c container-name -- /bin/sh
```

#### 常见错误及解决
```bash
# 错误：exec: "/bin/bash": no such file or directory
# 原因：容器没有 bash（如 Alpine Linux）

# 解决：使用 sh
kubectl exec -it pod-name -- /bin/sh

# 检查容器中有哪些 shell
kubectl exec pod-name -- ls /bin | grep sh

# 识别容器操作系统
kubectl exec pod-name -- cat /etc/os-release
```

#### 不同镜像的 Shell 情况
| 基础镜像 | 默认 Shell | 命令 |
|---------|-----------|------|
| alpine | /bin/sh | `kubectl exec -it pod -- sh` |
| ubuntu | /bin/bash | `kubectl exec -it pod -- bash` |
| debian | /bin/bash | `kubectl exec -it pod -- bash` |
| busybox | /bin/sh | `kubectl exec -it pod -- sh` |
| distroless | 无 shell | 使用 `kubectl debug` |

#### 调试没有 Shell 的容器
```bash
# 使用临时调试容器
kubectl debug pod-name -it --image=busybox

# 复制文件出来分析
kubectl cp pod-name:/app/config ./config
```

### 3. 通过 Service 访问

#### ClusterIP Service（仅集群内部）
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP  # 默认类型
```

#### NodePort Service（Docker Desktop 推荐）
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-nodeport
spec:
  type: NodePort
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080  # 30000-32767
```
```bash
# Docker Desktop 上访问
open http://localhost:30080
```

#### LoadBalancer Service（Docker Desktop 模拟）
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-lb
spec:
  type: LoadBalancer
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 80
```
```bash
# Docker Desktop 会将 EXTERNAL-IP 显示为 localhost
kubectl get svc my-lb
open http://localhost:80
```

### 4. 使用 kubectl proxy
```bash
# 启动代理
kubectl proxy

# 访问 Pod
curl http://localhost:8001/api/v1/namespaces/default/pods/my-pod/proxy/
```

## 调试 Pod

### 1. 查看日志
```bash
# 查看日志
kubectl logs my-pod

# 实时日志
kubectl logs -f my-pod

# 查看之前的日志（如果容器重启过）
kubectl logs my-pod --previous

# 多容器 Pod
kubectl logs my-pod -c container-name

# 最近 N 行
kubectl logs my-pod --tail=100

# 时间范围
kubectl logs my-pod --since=1h
```

### 2. 调试状态异常的 Pod

#### Pod 状态说明
- **Pending**: Pod 已创建但未调度或镜像未下载
- **Running**: Pod 正在运行
- **Succeeded**: Pod 成功完成（Job）
- **Failed**: Pod 失败
- **Unknown**: 无法获取 Pod 状态
- **CrashLoopBackOff**: 容器启动后崩溃，Kubernetes 正在重试

#### 常见调试命令
```bash
# 1. 查看 Pod 事件
kubectl describe pod my-pod

# 2. 查看日志
kubectl logs my-pod

# 3. 检查资源使用
kubectl top pod my-pod

# 4. 导出 Pod 定义进行分析
kubectl get pod my-pod -o yaml > pod-debug.yaml

# 5. 查看节点信息
kubectl get nodes
kubectl describe node <node-name>
```

### 3. 调试网络问题
```bash
# 测试 Pod 网络
kubectl exec my-pod -- ping google.com
kubectl exec my-pod -- nslookup kubernetes.default

# 查看 Pod IP
kubectl get pod my-pod -o jsonpath='{.status.podIP}'

# 测试服务连接
kubectl exec my-pod -- curl http://service-name
```

### 4. 调试存储问题
```bash
# 查看挂载的卷
kubectl describe pod my-pod | grep -A 10 "Volumes:"

# 检查卷内容
kubectl exec my-pod -- ls -la /mount/path

# 查看 PVC 状态
kubectl get pvc
```

## 常见问题与解决方案

### 1. ImagePullBackOff
**问题**：无法拉取镜像
```bash
# 诊断
kubectl describe pod my-pod | grep -A 5 "Events:"

# 解决方案
# - 检查镜像名称和标签
# - 检查镜像仓库认证
# - 使用 imagePullSecrets
```

### 2. CrashLoopBackOff
**问题**：容器启动后立即崩溃
```bash
# 查看日志
kubectl logs my-pod --previous

# 常见原因
# - 应用程序错误
# - 缺少必需的环境变量
# - 配置文件错误
# - 依赖服务不可用
```

### 3. Pending 状态
**问题**：Pod 无法调度
```bash
# 查看事件
kubectl describe pod my-pod

# 常见原因
# - 资源不足
# - 节点选择器不匹配
# - PVC 未绑定
```

### 4. 资源限制问题
```bash
# 查看资源使用
kubectl top pod my-pod

# 调整资源限制
kubectl set resources pod my-pod --limits=cpu=500m,memory=512Mi
```

### 5. Shell 访问错误
**问题**：exec: "/bin/bash": no such file or directory
```bash
# 错误命令
kubectl exec -it my-pod -- /bin/bash

# 解决方案
# 1. 使用 sh（大多数容器都有）
kubectl exec -it my-pod -- /bin/sh

# 2. 检查可用的 shell
kubectl exec my-pod -- ls /bin | grep sh

# 3. 检查操作系统
kubectl exec my-pod -- cat /etc/os-release

# 常见原因
# - Alpine Linux 没有 bash，只有 sh
# - Distroless 镜像没有任何 shell
# - 精简镜像只保留最小必需组件
```

## 最佳实践

### 1. YAML 文件编写
- ✅ 使用 2 个空格缩进
- ✅ 不使用制表符
- ✅ 添加有意义的标签
- ✅ 设置资源限制
- ✅ 配置健康检查

### 2. 镜像选择
- ✅ 使用特定版本标签，避免 latest
- ✅ 使用官方或可信的基础镜像
- ✅ 最小化镜像大小
- ✅ 定期更新镜像

### 3. 安全实践
- ✅ 不以 root 用户运行
- ✅ 使用只读文件系统（如果可能）
- ✅ 限制容器权限
- ✅ 使用 Network Policies

### 4. 监控和日志
- ✅ 配置适当的日志级别
- ✅ 使用结构化日志
- ✅ 设置资源监控告警
- ✅ 定期审查日志

## 实战案例：Fleetman Webapp

### 失败案例分析
```yaml
# 初始配置（失败）
apiVersion: v1
kind: Pod
metadata:
  name: fleetman-webapp
spec:
  containers:
  - name: webapp
    image: richardchesterwood/k8s-fleetman-webapp-angular:release1
    ports:
    - containerPort: 80
```

**失败原因**：
1. 缺少必需的环境变量
2. 依赖后端服务不存在

### 调试过程
```bash
# 1. 查看状态
kubectl get pod fleetman-webapp

# 2. 查看日志发现环境变量错误
kubectl logs fleetman-webapp
# Error: 'SPRING_PROFILES_ACTIVE' is undefined

# 3. 添加环境变量后仍然失败
# Error: host not found in upstream "fleetman-api-gateway"

# 4. 分析：这是微服务的一部分，需要完整的服务栈
```

### 解决方案
1. 部署完整的微服务栈
2. 或使用独立的应用进行测试

## 快速参考

### 常用命令速查
```bash
# Pod 操作
kubectl apply -f pod.yaml          # 创建/更新
kubectl get pods                   # 列出
kubectl describe pod <name>        # 详情
kubectl logs <pod>                 # 日志
kubectl exec -it <pod> -- sh       # 进入（推荐，更通用）
kubectl exec -it <pod> -- bash     # 进入（如果有bash）
kubectl port-forward <pod> 8080:80 # 端口转发
kubectl delete pod <name>          # 删除

# 调试命令
kubectl logs <pod> --previous      # 之前的日志
kubectl top pod <name>             # 资源使用
kubectl get events                 # 查看事件
```

### Docker Desktop 快速访问
```bash
# 端口转发
kubectl port-forward pod-name 8080:80
open http://localhost:8080

# NodePort Service
kubectl expose pod pod-name --type=NodePort --port=80
kubectl get svc pod-name  # 查看分配的端口
open http://localhost:3xxxx

# LoadBalancer Service (Docker Desktop 映射到 localhost)
kubectl expose pod pod-name --type=LoadBalancer --port=80
open http://localhost:80
```

### 状态诊断决策树
```
Pod 状态异常？
├── Pending
│   ├── 检查节点资源
│   ├── 检查 PVC
│   └── 检查调度约束
├── CrashLoopBackOff
│   ├── 查看日志
│   ├── 检查环境变量
│   └── 检查依赖服务
├── ImagePullBackOff
│   ├── 检查镜像名称
│   ├── 检查仓库认证
│   └── 检查网络连接
└── Running 但无法访问
    ├── 检查端口配置
    ├── 检查服务配置
    └── 检查网络策略
```

## 总结

成功运行 Pod 的关键：
1. 正确的 YAML 配置
2. 合适的镜像选择
3. 必需的环境变量
4. 足够的资源分配
5. 正确的网络配置
6. 有效的调试方法

记住：**日志是你最好的朋友！**
