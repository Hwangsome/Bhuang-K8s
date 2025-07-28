# Kubernetes 集群备份和恢复方法

## 1. ETCD 集群访问

### 1.1 查找 ETCD 地址

首先需要在控制平面节点上找到 ETCD 集群的访问地址。可以通过检查 ETCD Pod 的配置来获取这些信息：

```bash
# 查找 ETCD Pod
kubectl -n kube-system get pods -l component=etcd

# 获取 ETCD Pod 名称（用于后续命令）
ETCD_POD=$(kubectl -n kube-system get pods -l component=etcd -o jsonpath='{.items[0].metadata.name}')

# 查看 ETCD Pod 的详细配置
kubectl -n kube-system describe pod $ETCD_POD
```

### 1.2 ETCD 访问地址

在控制平面节点上，可以通过以下地址访问 ETCD 集群：

1. 客户端访问地址（`--advertise-client-urls` 和 `--listen-client-urls`）：
   - `https://127.0.0.1:2379`（本地回环地址）
   - `https://192.168.65.4:2379`（节点 IP 地址）

2. 对等节点通信地址（`--listen-peer-urls`）：
   - `https://192.168.65.4:2380`

### 1.3 ETCD 安全访问配置

ETCD 使用 HTTPS 进行安全通信，需要使用证书进行认证。证书文件位置：
- 证书目录：`/run/config/pki/etcd/`
- CA 证书：`ca.crt`
- 服务器证书：`server.crt`
- 服务器密钥：`server.key`

### 1.4 使用 etcdctl 访问 ETCD

要使用 `etcdctl` 命令行工具访问 ETCD，需要设置以下环境变量：

```bash
export ETCDCTL_API=3
export ETCDCTL_CACERT=/run/config/pki/etcd/ca.crt
export ETCDCTL_CERT=/run/config/pki/etcd/server.crt
export ETCDCTL_KEY=/run/config/pki/etcd/server.key
export ETCDCTL_ENDPOINTS=https://127.0.0.1:2379
```

## 2. ETCD 备份

### 2.1 创建快照备份

使用 `etcdctl snapshot save` 命令创建 ETCD 数据的快照：

```bash
ETCDCTL_API=3 etcdctl snapshot save snapshot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/run/config/pki/etcd/ca.crt \
  --cert=/run/config/pki/etcd/server.crt \
  --key=/run/config/pki/etcd/server.key
```

### 2.2 验证快照

使用 `etcdctl snapshot status` 命令验证快照：

```bash
ETCDCTL_API=3 etcdctl snapshot status snapshot.db
```

## 3. ETCD 恢复

### 3.1 停止 Kubernetes API 服务器

```bash
systemctl stop kube-apiserver
```

### 3.2 从快照恢复

使用 `etcdctl snapshot restore` 命令从快照恢复数据：

```bash
ETCDCTL_API=3 etcdctl snapshot restore snapshot.db \
  --data-dir=/var/lib/etcd-backup \
  --initial-cluster=docker-desktop=https://192.168.65.4:2380 \
  --initial-advertise-peer-urls=https://192.168.65.4:2380 \
  --name=docker-desktop
```

### 3.3 更新 ETCD 配置

1. 修改 ETCD 静态 Pod 配置，指向新的数据目录
2. 重启 ETCD Pod

### 3.4 重启 Kubernetes API 服务器

```bash
systemctl start kube-apiserver
```

## 4. 最佳实践

1. **定期备份**：
   - 建立定期备份计划
   - 保留多个备份版本
   - 验证备份的完整性

2. **安全存储**：
   - 将备份存储在安全的位置
   - 加密备份数据
   - 定期测试恢复过程

3. **文档化**：
   - 记录备份和恢复程序
   - 维护证书和密钥的位置信息
   - 记录 ETCD 集群配置

4. **监控**：
   - 监控 ETCD 集群健康状态
   - 监控备份作业的执行情况
   - 设置告警机制

## 5. 故障排除

1. **证书问题**：
   - 确保证书路径正确
   - 验证证书的有效期
   - 检查证书权限

2. **网络问题**：
   - 验证 ETCD 端点可访问性
   - 检查防火墙规则
   - 确认网络策略配置

3. **存储问题**：
   - 确保有足够的磁盘空间
   - 检查数据目录权限
   - 监控磁盘性能

4. **版本兼容性**：
   - 确保 etcdctl 版本与 ETCD 服务器版本匹配
   - 检查 Kubernetes 版本兼容性
   - 注意升级时的版本跨度