# Docker Volume 使用指南

## 1. 项目结构

```
code/
├── docker-volume-demo.yaml    # Docker Compose 配置文件
├── config/
│   └── default.conf          # Nginx 配置
├── mysql/
│   ├── conf/
│   │   └── my.cnf           # MySQL 配置
│   └── init/
│       └── init.sql         # MySQL 初始化脚本
└── index.html               # 测试网页
```

## 2. 配置文件说明

### 2.1 Docker Compose 配置

`docker-volume-demo.yaml` 定义了三个服务：

1. **Web 服务 (Nginx)**
   - 使用命名卷存储网站内容
   - 使用绑定挂载存储配置文件
   - 暴露 8080 端口

2. **数据库服务 (MySQL)**
   - 使用命名卷存储数据
   - 使用绑定挂载存储配置和初始化脚本
   - 暴露 3306 端口

3. **测试服务 (Alpine)**
   - 使用临时卷
   - 用于测试目的

### 2.2 Volume 类型演示

1. **命名卷 (Named Volumes)**
   - `web_data_volume`: 存储网站内容
   - `db_data_volume`: 存储数据库数据

2. **绑定挂载 (Bind Mounts)**
   - `./config`: Nginx 配置
   - `./mysql/conf`: MySQL 配置
   - `./mysql/init`: MySQL 初始化脚本

3. **临时卷 (Anonymous Volumes)**
   - `/tmp/test`: 临时存储

## 3. 部署和验证步骤

### 3.1 启动服务

```bash
# 启动所有服务
docker compose -f docker-volume-demo.yaml up -d
```

### 3.2 验证部署

1. **检查容器状态**
```bash
docker compose -f docker-volume-demo.yaml ps
```

2. **验证 Volume 创建**
```bash
docker volume ls | grep 'web_data\|db_data'
```

3. **测试 Web 服务**
```bash
# 访问网页
curl http://localhost:8080

# 更新网页内容
docker cp index.html code-web-1:/usr/share/nginx/html/

# 验证更新
curl http://localhost:8080
```

4. **验证数据库服务**
```bash
# 查询测试数据
docker exec code-db-1 mysql -u testuser -ptestpass -e "USE testdb; SELECT * FROM users;"
```

### 3.3 数据持久性测试

1. **停止并删除容器**
```bash
docker compose -f docker-volume-demo.yaml down
```

2. **重新创建容器**
```bash
docker compose -f docker-volume-demo.yaml up -d
```

3. **验证数据持久性**
```bash
# 等待数据库启动
sleep 10

# 验证数据是否存在
docker exec code-db-1 mysql -u testuser -ptestpass -e "USE testdb; SELECT * FROM users;"
```

## 4. 清理环境

```bash
# 停止并删除容器
docker compose -f docker-volume-demo.yaml down

# 删除 volumes（可选，会删除所有数据）
docker compose -f docker-volume-demo.yaml down -v
```

## 5. 验证结果

通过以上步骤的验证，我们可以确认：

1. **容器正常运行**：所有服务（web、db、test）都正常启动和运行。

2. **Volume 正确创建**：成功创建并使用了命名卷。

3. **Web 服务功能**：
   - 成功访问 Nginx 服务
   - 成功更新和持久化网页内容

4. **数据库服务功能**：
   - MySQL 正确初始化
   - 测试数据成功导入
   - 数据在容器重启后保持完整

5. **数据持久性**：
   - 容器删除后数据保持完整
   - 重建容器后数据自动恢复

## 6. 最佳实践

1. **使用命名卷**：对重要数据使用命名卷，便于管理和备份。

2. **配置文件使用绑定挂载**：便于直接编辑和版本控制。

3. **初始化脚本**：使用 `docker-entrypoint-initdb.d` 自动初始化数据库。

4. **环境变量**：敏感信息使用环境变量或 secrets 管理。

5. **权限管理**：正确设置文件权限和用户权限。

## 7. 故障排除

如果遇到问题，可以：

1. 检查容器日志：
```bash
docker compose -f docker-volume-demo.yaml logs
```

2. 检查 volume 内容：
```bash
docker volume inspect web_data_volume
docker volume inspect db_data_volume
```

3. 验证文件权限：
```bash
docker exec -it code-web-1 ls -la /usr/share/nginx/html
docker exec -it code-db-1 ls -la /var/lib/mysql
```

# Kubernetes Volumes 使用指南

本章节演示了在 Kubernetes 中使用不同类型的存储卷的配置和验证方法。

## 1. 存储卷类型

在示例中，我们将展示三种常用的存储卷类型：

1. **PersistentVolume (PV) 和 PersistentVolumeClaim (PVC)**
   - 用于持久化存储
   - 支持数据持久性
   - 独立于 Pod 生命周期

2. **ConfigMap**
   - 用于配置文件
   - 可以动态更新
   - 支持多个 Pod 共享配置

3. **EmptyDir**
   - 临时存储
   - Pod 内容器间共享数据
   - Pod 删除时数据丢失

## 2. 配置说明

### 2.1 PV 和 PVC 配置

```yaml
# PersistentVolume 配置
apiVersion: v1
kind: PersistentVolume
metadata:
  name: example-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /data/example-pv

# PersistentVolumeClaim 配置
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: example-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

### 2.2 在 Pod 中使用存储卷

```yaml
spec:
  containers:
  - name: nginx
    volumeMounts:
    - name: nginx-data
      mountPath: /usr/share/nginx/html
    - name: nginx-config
      mountPath: /etc/nginx/conf.d
  volumes:
  - name: nginx-data
    persistentVolumeClaim:
      claimName: example-pvc
  - name: nginx-config
    configMap:
      name: nginx-config
```

## 3. 部署和验证步骤

### 3.1 创建资源

```bash
# 应用配置
kubectl apply -f k8s-volumes.yaml

# 等待 Pod 就绪
kubectl wait --for=condition=Ready pod -l app=nginx
```

### 3.2 验证部署

1. **检查 PV 和 PVC 状态**
```bash
kubectl get pv,pvc
```

2. **验证 Pod 状态**
```bash
kubectl get pods
kubectl describe pod nginx-deployment-xxx
```

3. **测试 Web 服务**
```bash
# 访问服务
curl http://localhost:30080

# 查看 Pod 日志
kubectl logs -l app=nginx
```

4. **验证 ConfigMap**
```bash
# 查看 ConfigMap 内容
kubectl get configmap nginx-config -o yaml

# 检查配置是否正确挂载
kubectl exec -it <pod-name> -- ls -l /etc/nginx/conf.d
```

5. **测试 EmptyDir**
```bash
# 查看共享卷中的内容
kubectl exec -it shared-volumes-pod -c nginx-container -- cat /usr/share/nginx/html/index.html
```

### 3.3 数据持久性测试

1. **删除并重建 Pod**
```bash
# 删除 Pod
kubectl delete pod -l app=nginx

# 等待新 Pod 创建
kubectl wait --for=condition=Ready pod -l app=nginx

# 验证数据是否保持
curl http://localhost:30080
```

## 4. 清理资源

```bash
# 删除所有资源
kubectl delete -f k8s-volumes.yaml
```

## 5. 最佳实践

1. **存储规划**
   - 根据需求选择适当的存储类型
   - 考虑数据持久性要求
   - 规划适当的存储容量

2. **安全性考虑**
   - 使用适当的访问模式
   - 配置存储类的回收策略
   - 实施适当的备份策略

3. **性能优化**
   - 选择合适的存储类
   - 合理设置资源限制
   - 监控存储使用情况

4. **维护建议**
   - 定期监控存储使用情况
   - 实施备份策略
   - 及时清理不需要的数据

## 6. 故障排除

如果遇到问题，可以：

1. **检查 PV/PVC 状态**
```bash
kubectl describe pv example-pv
kubectl describe pvc example-pvc
```

2. **检查 Pod 事件**
```bash
kubectl describe pod <pod-name>
```

3. **查看 Pod 日志**
```bash
kubectl logs <pod-name>
```

4. **验证挂载点**
```bash
kubectl exec -it <pod-name> -- df -h
kubectl exec -it <pod-name> -- mount | grep <mount-path>
