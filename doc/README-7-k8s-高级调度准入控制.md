# 高级调度准入控制
## Resource Quota
### Kubernetes 的 ResourceQuota 概述

**ResourceQuota** 是 Kubernetes 中的一种命名空间级别的资源管理策略，用于限制特定命名空间中资源的总使用量。通过 ResourceQuota，可以有效控制命名空间的资源分配，防止某些工作负载或用户独占集群资源。

---

### **ResourceQuota 的作用**

1. **限制资源使用量**
    - 防止命名空间中的资源超出指定的配额，例如 CPU、内存、存储等。
    - 限制对象的数量，如 Pod、Service、ConfigMap 等。

2. **实现多租户资源隔离**
    - 在多租户环境中，防止一个租户耗尽整个集群的资源。
    - 为不同的团队或应用分配资源上限。

3. **与 LimitRange 配合**
    - ResourceQuota 可与 LimitRange 配合，强制 Pod 和容器必须设置资源请求和限制。

---

### **ResourceQuota 的配置结构**

一个典型的 `ResourceQuota` 配置如下：

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: example-quota
  namespace: my-namespace
spec:
  hard:
    pods: "10"                      # 限制最多 10 个 Pod
    requests.cpu: "2"               # CPU 请求总量不能超过 2 核
    requests.memory: "4Gi"          # 内存请求总量不能超过 4Gi
    limits.cpu: "4"                 # CPU 限制总量不能超过 4 核
    limits.memory: "8Gi"            # 内存限制总量不能超过 8Gi
    persistentvolumeclaims: "5"     # 限制最多 5 个 PVC
    services: "10"                  # 限制最多 10 个 Service
    configmaps: "20"                # 限制最多 20 个 ConfigMap
    secrets: "10"                   # 限制最多 10 个 Secret
    storage.requests: "100Gi"       # 存储请求总量不能超过 100Gi
```

---

### **常用字段解释**

- **`spec.hard`**: 定义硬性限制的资源配额，包括资源的类型和上限。
    - 计算资源：`requests.cpu`、`limits.cpu`、`requests.memory`、`limits.memory`
    - 存储资源：`persistentvolumeclaims`、`storage.requests`
    - 对象计数：`pods`、`services`、`configmaps`、`secrets`、`replicationcontrollers`

- **`metadata.namespace`**: 表示该配额适用于哪个命名空间。

---

### **资源配额的工作原理**

1. **创建 ResourceQuota**
    - ResourceQuota 是命名空间级别的对象，因此需要在特定命名空间中创建。

2. **资源使用情况监控**
    - Kubernetes 的控制器会定期检查命名空间内资源的使用情况，并与 ResourceQuota 的限制进行比较。

3. **拒绝超额请求**
    - 当用户尝试创建超出配额的资源时，请求会被拒绝，并返回错误信息。

4. **动态调整**
    - 可以随时更新 ResourceQuota 的配置，动态调整配额。

---

### **验证 ResourceQuota**

1. **查看已配置的 ResourceQuota**
   ```bash
   kubectl get resourcequota -n my-namespace
   ```

2. **详细查看 ResourceQuota 配置**
   ```bash
   kubectl describe resourcequota example-quota -n my-namespace
   ```

3. **查看命名空间资源使用情况**
   ```bash
   kubectl get resourcequota example-quota -n my-namespace --output=yaml
   ```

---

### **与 LimitRange 的结合**

**LimitRange** 用于设置单个容器或 Pod 的资源请求和限制，而 ResourceQuota 管理的是整个命名空间的资源总量。

例如：
- ResourceQuota 强制命名空间中的资源总量不超过 10 个 CPU。
- LimitRange 确保单个 Pod 不会申请超过 2 个 CPU。

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: pod-limit-range
  namespace: my-namespace
spec:
  limits:
    - type: Container
      max:
        cpu: "2"
        memory: "2Gi"
      min:
        cpu: "200m"
        memory: "256Mi"
```

---

### **常见问题和注意事项**

1. **默认情况下，未设置 ResourceQuota 的命名空间无资源限制**。
    - 如果需要全局资源控制，需要在所有命名空间中设置 ResourceQuota。

2. **创建资源时必须指定资源请求和限制**。
    - 如果 LimitRange 配置要求 Pod 必须指定资源请求和限制，未设置的 Pod 会被拒绝。

3. **与动态存储卷的结合**。
    - 配额中设置的 `storage.requests` 会影响 PersistentVolumeClaim 的动态创建。

4. **适配多租户需求**。
    - 在多租户环境中，ResourceQuota 和 LimitRange 是资源隔离的基础配置。

---

### **实用示例**

#### 限制 Pod 和存储资源

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: storage-quota
  namespace: team-namespace
spec:
  hard:
    pods: "50"                # 限制最多 50 个 Pod
    storage.requests: "500Gi" # 限制存储请求总量为 500Gi
```

#### 限制对象数量

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: object-count-quota
  namespace: app-namespace
spec:
  hard:
    configmaps: "10"          # 限制最多 10 个 ConfigMap
    secrets: "10"             # 限制最多 10 个 Secret
    services: "5"             # 限制最多 5 个 Service
```

---

## LimitRange 
### Kubernetes 中的 LimitRange 详解

**LimitRange** 是 Kubernetes 中的一种资源策略，用于限制单个容器、Pod 或其他资源对象（如 PersistentVolumeClaim）的资源请求（`requests`）和限制（`limits`）。它的作用是确保每个工作负载的资源使用符合预期，从而实现资源的合理分配和保护集群的稳定性。

---

### **LimitRange 的作用**

1. **设置资源最小值和最大值**
   - 为容器或 Pod 定义允许的最小和最大资源请求与限制。
   - 确保某些应用不会因未设置资源而无限制使用。

2. **默认资源分配**
   - 如果容器未指定 `requests` 和 `limits`，LimitRange 可以为其设置默认值。

3. **资源保护和公平分配**
   - 限制单个工作负载对资源的滥用。
   - 在多租户环境中，保障不同应用的资源公平性。

4. **限制存储请求**
   - 对 PersistentVolumeClaim（PVC）设置最小和最大存储请求。

---

### **LimitRange 的配置结构**

以下是一个典型的 LimitRange 示例：

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: example-limits
  namespace: my-namespace
spec:
  limits:
    - type: Container                    # 限制作用于容器
      max:
        cpu: "2"                         # 最大允许使用 2 个 CPU 核心
        memory: "1Gi"                    # 最大允许使用 1Gi 内存
      min:
        cpu: "200m"                      # 最小需要 200 毫核（0.2 个核心）
        memory: "128Mi"                  # 最小需要 128Mi 内存
      default:
        cpu: "1"                         # 未设置时默认分配 1 个 CPU 核心
        memory: "512Mi"                  # 未设置时默认分配 512Mi 内存
      defaultRequest:
        cpu: "500m"                      # 未设置时请求值默认 500 毫核
        memory: "256Mi"                  # 未设置时请求值默认 256Mi 内存
```

---

### **配置字段解释**

- **`type`**：
   - 指定限制的作用范围：
      - `Container`：作用于容器。
      - `Pod`：作用于整个 Pod。
      - `PersistentVolumeClaim`：作用于存储请求（PVC）。

- **`max`**：
   - 定义允许的资源最大值。

- **`min`**：
   - 定义允许的资源最小值。

- **`default`**：
   - 如果容器未设置资源限制，则使用此值作为默认 `limits`。

- **`defaultRequest`**：
   - 如果容器未设置资源请求，则使用此值作为默认 `requests`。

- **`maxLimitRequestRatio`**：
   - 定义 `limits` 与 `requests` 的最大比例，限制资源请求与限制的差异范围。

---

### **LimitRange 的工作原理**

1. **创建 LimitRange 对象**
   - LimitRange 是一个命名空间级别的对象。
   - 创建后，命名空间内的新资源（如 Pod、容器）会自动应用此限制。

2. **验证资源定义**
   - 在创建 Pod 或容器时，Kubernetes 会验证其资源定义是否符合 LimitRange 的要求。
   - 不符合要求的请求会被拒绝。

3. **默认资源分配**
   - 如果 Pod 或容器未指定 `requests` 或 `limits`，则 Kubernetes 使用 LimitRange 中的 `default` 和 `defaultRequest` 自动设置。

---

### **验证 LimitRange**

1. **查看已配置的 LimitRange**
   ```bash
   kubectl get limitrange -n my-namespace
   ```

2. **查看 LimitRange 详情**
   ```bash
   kubectl describe limitrange example-limits -n my-namespace
   ```

---

### **LimitRange 的示例配置**

#### 示例 1：限制容器的 CPU 和内存
```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: container-resource-limits
  namespace: my-namespace
spec:
  limits:
    - type: Container
      max:
        cpu: "2"
        memory: "1Gi"
      min:
        cpu: "100m"
        memory: "64Mi"
      default:
        cpu: "1"
        memory: "512Mi"
      defaultRequest:
        cpu: "500m"
        memory: "256Mi"
```

- **解释**：
   - 每个容器必须至少请求 100m CPU 和 64Mi 内存，最多使用 2 CPU 和 1Gi 内存。
   - 如果未显式指定 `requests` 和 `limits`，容器默认请求 500m CPU、256Mi 内存，限制为 1 CPU、512Mi 内存。

#### 示例 2：限制 Pod 的总资源
```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: pod-resource-limits
  namespace: my-namespace
spec:
  limits:
    - type: Pod
      max:
        cpu: "4"
        memory: "4Gi"
      min:
        cpu: "200m"
        memory: "256Mi"
```

- **解释**：
   - 每个 Pod 的资源总量必须至少请求 200m CPU 和 256Mi 内存，最多使用 4 CPU 和 4Gi 内存。

#### 示例 3：限制存储请求
```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: pvc-storage-limits
  namespace: my-namespace
spec:
  limits:
    - type: PersistentVolumeClaim
      max:
        storage: "100Gi"
      min:
        storage: "1Gi"
```

- **解释**：
   - 每个 PVC 的存储请求必须在 1Gi 到 100Gi 之间。

---

### **与 ResourceQuota 的关系**

- **LimitRange**：
   - 用于控制单个资源（容器、Pod、PVC）的请求和限制。
   - 强制工作负载必须符合资源使用规则。

- **ResourceQuota**：
   - 用于限制整个命名空间的资源总量。
   - 控制某个命名空间中所有工作负载的资源使用上限。

#### 配合使用场景：
- **LimitRange** 确保单个 Pod 或容器不会滥用资源。
- **ResourceQuota** 确保整个命名空间不会超出分配的资源上限。

---

### **常见问题和注意事项**

1. **未设置 LimitRange 时的默认行为**
   - 如果未设置 LimitRange 且 Pod 未指定资源请求和限制，Pod 可以使用节点上的全部剩余资源。

2. **LimitRange 不能跨命名空间生效**
   - 每个 LimitRange 只能影响所在命名空间内的资源。

3. **强制性规则**
   - Pod 或容器必须满足 LimitRange 的 `min` 和 `max`，否则创建请求会被拒绝。

4. **合理的默认值**
   - 设置适合业务场景的 `default` 和 `defaultRequest`，减少开发人员手动配置的工作量。

---

### 总结

**LimitRange** 是 Kubernetes 中管理资源的重要工具，通过定义最小、最大、默认资源限制，帮助实现资源的合理分配和集群的稳定运行。合理配置 LimitRange 可以在多租户和多应用场景中避免资源竞争，提高集群的可靠性和性能。


---

## QOS
在 Kubernetes 中，**QoS（Quality of Service，服务质量）** 是一种机制，用于根据 Pod 的资源请求和限制，为其分配优先级，确保在资源争夺和节点资源不足的情况下，提供合理的服务质量保障。

Kubernetes 使用 QoS 类别（QoS Class）对 Pod 进行分类，主要分为三种：**Guaranteed**、**Burstable** 和 **BestEffort**。

---

### **QoS 的分类**

Kubernetes 根据 Pod 中容器的资源配置，将其分配到以下 QoS 类别：

#### **1. Guaranteed**
- **定义**：
    - 所有容器的资源 **`requests` 和 `limits` 都必须设置**，且两者的值必须相等。
- **优点**：
    - Guaranteed 类型的 Pod 拥有最高优先级，在节点资源不足时，Guaranteed Pod 最后才会被驱逐。
- **适用场景**：
    - 关键业务场景，需要确保服务稳定运行。
- **配置示例**：
  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: guaranteed-pod
  spec:
    containers:
      - name: app
        image: nginx
        resources:
          requests:
            cpu: "1"
            memory: "500Mi"
          limits:
            cpu: "1"
            memory: "500Mi"
  ```
    - 在该 Pod 中，`requests` 和 `limits` 完全相等，因此它属于 **Guaranteed**。

---

#### **2. Burstable**
- **定义**：
    - 至少一个容器设置了 `requests`，但并非所有容器的 `requests` 和 `limits` 都相等。
    - 如果未设置 `limits`，Kubernetes 会默认允许容器使用节点上的所有剩余资源。
- **优点**：
    - 在资源充足时，Pod 可灵活使用多余资源。
    - 优先级高于 BestEffort，但低于 Guaranteed。
- **适用场景**：
    - 需要一定资源保障，但不要求严格限制的场景。
- **配置示例**：
  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: burstable-pod
  spec:
    containers:
      - name: app
        image: nginx
        resources:
          requests:
            cpu: "500m"
            memory: "256Mi"
          limits:
            cpu: "1"
            memory: "512Mi"
  ```
    - 在该 Pod 中，`requests` 和 `limits` 不相等，因此它属于 **Burstable**。

---

#### **3. BestEffort**
- **定义**：
    - 所有容器都没有设置任何资源 `requests` 和 `limits`。
- **优点**：
    - BestEffort 类型的 Pod 可以尽可能利用节点的空闲资源。
- **缺点**：
    - 优先级最低，在资源不足时，这类 Pod 最先被驱逐。
- **适用场景**：
    - 测试、开发环境或非关键性任务。
- **配置示例**：
  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: besteffort-pod
  spec:
    containers:
      - name: app
        image: nginx
  ```
    - 该 Pod 未设置任何资源 `requests` 和 `limits`，因此它属于 **BestEffort**。

---

### **QoS 类别的优先级**

在资源不足或节点压力过大时，Kubernetes 会根据 QoS 类别决定驱逐优先级：

1. **Guaranteed**：优先级最高，最后被驱逐。
2. **Burstable**：优先级中等。
3. **BestEffort**：优先级最低，最先被驱逐。

---

### **QoS 的工作机制**

1. **Pod 调度阶段**：
    - Kubernetes 调度器根据 Pod 的 `requests` 确定其可以调度到的节点。
    - Pod 的 QoS 类别不会影响调度优先级，但影响运行时的资源竞争行为。

2. **运行时资源分配**：
    - Guaranteed Pod 始终可以获得其请求的资源，且限制其使用范围。
    - Burstable Pod 在满足 `requests` 的前提下，可以争抢节点的多余资源。
    - BestEffort Pod 只能使用节点的空闲资源。

3. **节点资源不足时**：
    - Kubernetes 使用 **驱逐机制（Eviction Policy）** 释放资源。
    - 根据 QoS 类别，优先驱逐 BestEffort 类型的 Pod。

---

### **QoS 示例对比**

#### **示例 1：Guaranteed**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: guaranteed-pod
spec:
  containers:
    - name: app
      image: nginx
      resources:
        requests:
          cpu: "1"
          memory: "1Gi"
        limits:
          cpu: "1"
          memory: "1Gi"
```
- **分类**：Guaranteed
- **行为**：
    - 总是能获得 1 个 CPU 和 1Gi 内存。
    - 不会使用超过 1 个 CPU 和 1Gi 内存。

---

#### **示例 2：Burstable**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: burstable-pod
spec:
  containers:
    - name: app
      image: nginx
      resources:
        requests:
          cpu: "500m"
          memory: "256Mi"
        limits:
          cpu: "2"
          memory: "1Gi"
```
- **分类**：Burstable
- **行为**：
    - 总是能获得 500m CPU 和 256Mi 内存。
    - 如果有多余资源，最多可以使用 2 个 CPU 和 1Gi 内存。

---

#### **示例 3：BestEffort**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: besteffort-pod
spec:
  containers:
    - name: app
      image: nginx
```
- **分类**：BestEffort
- **行为**：
    - 不保证任何资源。
    - 只能使用节点的空闲资源。

---

### **QoS 的重要性**

1. **资源争夺时的公平性**
    - 确保重要服务不会因资源不足而中断。
    - 限制非关键性任务对资源的无限制占用。

2. **提高集群稳定性**
    - QoS 分类帮助 Kubernetes 更好地管理资源，避免资源过载。

3. **与 LimitRange 和 ResourceQuota 配合**
    - LimitRange 可以强制 Pod 设置资源请求和限制，从而提升 QoS 等级。
    - ResourceQuota 限制命名空间的资源总量，间接影响 QoS 的分类。

---

### **最佳实践**

1. **总是为关键任务设置 `requests` 和 `limits`**
    - 保证其能够在资源争夺中稳定运行。
    - 提升 QoS 至 **Guaranteed**。

2. **开发环境可以使用 BestEffort**
    - 避免占用过多资源，最大化利用集群的空闲资源。

3. **对非关键任务使用 Burstable**
    - 设置合适的 `requests`，确保基本资源需求。
    - 允许弹性使用多余资源。

4. **与节点资源结合**
    - 根据节点的实际资源配置，合理设置 `requests` 和 `limits`，避免 Pod 调度失败。
