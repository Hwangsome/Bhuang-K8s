# Service 示例文件目录

本目录包含各种 Kubernetes Service 配置的 YAML 示例文件。

## 文件列表

- `clusterip-service.yaml` - ClusterIP 类型 Service 示例
- `nodeport-service.yaml` - NodePort 类型 Service 示例  
- `loadbalancer-service.yaml` - LoadBalancer 类型 Service 示例
- `externalname-service.yaml` - ExternalName 类型 Service 示例
- `headless-service.yaml` - Headless Service 示例
- `multi-port-service.yaml` - 多端口 Service 配置示例
- `session-affinity-service.yaml` - 会话亲和性配置示例

## 使用方法

```bash
# Apply a service configuration
kubectl apply -f <filename>.yaml

# Check service status
kubectl get service <service-name>

# Delete a service
kubectl delete -f <filename>.yaml
```

每个 YAML 文件都包含详细的注释，解释了各个配置项的作用和使用场景。
