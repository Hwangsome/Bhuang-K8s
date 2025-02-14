
# service 域名
在 Kubernetes 中，Service（简称 SVC）是一种抽象方式，用于定义一组 Pods 的访问策略。每个 Service 都可以有一个 DNS 名称，便于在集群内和外部访问。
---
![img_5.png](img%2Fimg_5.png)

### 1. Service 的基本概念

Kubernetes 中的 Service 提供了一种稳定的方式来访问一组后端 Pods。Pods 是动态的，可能会随着时间的推移而增加、减少或替换，而 Service 则通过提供一个虚拟 IP 和 DNS 名称来保持对这些 Pods 的一致访问。

### 2. Service 的类型

Kubernetes 提供了多种类型的 Service，每种类型的行为略有不同：

- **ClusterIP**：默认类型，只能在集群内部访问。Service 会被分配一个虚拟 IP 地址（Cluster IP），所有在同一个命名空间内的 Pods 都可以通过这个 IP 地址访问 Service。

- **NodePort**：在 ClusterIP 的基础上，额外提供了一个端口（NodePort），允许从集群外部通过任意节点的 IP 和该端口访问 Service。

- **LoadBalancer**：在云环境中，LoadBalancer 类型的 Service 会创建一个外部负载均衡器，用户可以通过负载均衡器的地址来访问服务。前端访问与后端的 ClusterIP 和 Pods 连接。

- **ExternalName**：允许将 Service 映射到外部的 DNS 名称，而不是 Pods。提供了对外部服务的简单访问。

### 3. Service 的 DNS 名称

在 Kubernetes 中，Service 会自动得到一个 DNS 名称。这个 DNS 名称由以下部分组成：

- **服务名称**：Service 的名称。
- **命名空间**：Service 所属的命名空间。
- **集群域名**：由 Kubernetes 集群设置的 DNS 后缀，通常是 `cluster.local`。

#### 3.1 DNS 名称格式

Service 的 DNS 名称的标准格式如下：

```
<service-name>.<namespace>.svc.<cluster-domain>
```
例如，如果有一个名为 `my-service` 的 Service 在 `default` 命名空间中，而集群的域名是 `cluster.local`，那么对应的 DNS 名称为：

```
my-service.default.svc.cluster.local
```

### 4. 访问 Service

在 Kubernetes 中，Pods 可以通过 DNS 名称直接访问 Service。例如，如果在 `my-pod` 中需要访问上面的 `my-service`，可以使用下面的命令：

```bash
curl http://my-service.default.svc.cluster.local
```

还可以直接使用部分 DNS 名称，例如：

```bash
curl http://my-service.default
```

Kubernetes 的 DNS 系统会自动解析为完整的 Service 地址。

### 5. 使用 Headless Service

如果不想为 Service 分配 Cluster IP，可以创建一个 "headless Service"。这种情况下，会在 Service 的定义中将 `clusterIP` 设置为 `None`。Headless Service 允许客户端直接与后端 Pod 进行通信，常用于状态服务（如数据库）等场景。

#### 示例配置：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-headless-service
spec:
  clusterIP: None
  selector:
    app: my-app
  ports:
  - port: 80
```

### 6. 总结

Kubernetes 中的 Service 提供了一个稳定的方式来访问和管理 Pods，确保应用在动态环境中的一致性。通过自动生成的 DNS 名称，能够轻松访问 Service，并且能够支持多种服务类型，适应不同的访问需求。Service 的 DNS 特性使得在微服务架构中，可以简化服务间的调用、减少配置麻烦，提升系统整体的可维护性与可扩展性。

# service 域名测试
## 使用test-pod.yaml 创建一个容器
```shell
kubectl apply -f test-pod.yaml
# 等待 Pod 运行起来：
kubectl wait --for=condition=Ready pod/test-busybox
# 使用完整域名测试：
kubectl exec -it test-busybox -- wget -qO- http://frontend-service.default.svc.cluster.local
# 使用短域名测试：
kubectl exec -it test-busybox -- wget -qO- http://frontend-service
#  使用 nslookup 测试域名解析：
kubectl exec -it test-busybox -- nslookup frontend-service.default.svc.cluster.local
# 使用 ClusterIP 测试：
kubectl exec -it test-busybox -- wget -qO- http://10.110.219.84

# 总命令
echo "=== 测试完整域名 ===" && kubectl exec -it test-busybox -- wget -qO- http://frontend-service.default.svc.cluster.local && echo -e "\n=== 测试短域名 ===" && kubectl exec -it test-busybox -- wget -qO- http://frontend-service && echo -e "\n=== 测试域名解析 ===" && kubectl exec -it test-busybox -- nslookup frontend-service.default.svc.cluster.local

```