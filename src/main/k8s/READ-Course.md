# Course
在 Kubernetes（K8s）中，定义资源的 YAML 文件时，确实始终包含四个顶级字段，它们是：

1. **`apiVersion`**：指定 Kubernetes 资源所属的 API 版本。
2. **`kind`**：指定要创建的 Kubernetes 资源类型，如 Pod、Service、Deployment 等。
3. **`metadata`**：包含资源的元数据，例如名称、命名空间和标签等。
4. **`spec`**：详细描述资源的期望状态（即声明性配置），例如 Pod 需要运行的容器、Deployment 的副本数等。

下面详细解释这四个字段：

---

### **1. apiVersion**

- **作用**：指定 Kubernetes 资源所使用的 API 版本。不同的 Kubernetes 资源属于不同的 API 组，每个 API 组中可能存在多个版本，常见的 API 版本有 `v1`、`apps/v1` 等。
- **示例**：`apiVersion: v1` 或 `apiVersion: apps/v1`。

- **常见版本**：
    - `v1`：用于核心 API 资源，如 Pod、Service、ConfigMap、Secret 等。
    - `apps/v1`：用于更复杂的工作负载资源，如 Deployment、DaemonSet、StatefulSet 等。

**示例：**

```yaml
apiVersion: apps/v1
```

---

### **2. kind**

- **作用**：指定要创建的 Kubernetes 资源的类型（对象的种类）。Kubernetes 中有许多不同的资源类型，例如 Pod、Service、Deployment、ConfigMap 等。
- **示例**：`kind: Pod` 或 `kind: Deployment`。

- **常见类型**：
    - `Pod`：定义一组容器的运行实例。
    - `Service`：用于将一组 Pod 暴露给其他服务或外部网络。
    - `Deployment`：用于管理 Pod 副本的控制器，支持滚动更新和回滚。
    - `ConfigMap`：用于管理配置数据。

**示例：**

```yaml
kind: Deployment
```

---

### **3. metadata**

- **作用**：包含 Kubernetes 资源的元数据信息，例如资源的名称、命名空间、标签和注解。
- **常用字段**：
    - **name**：资源的名称，必须是集群中唯一的。
    - **namespace**：资源所属的命名空间（可选，如果不指定则默认是 `default` 命名空间）。
    - **labels**：用于对资源进行标记，可以通过标签选择器筛选资源。
    - **annotations**：存储非标记性元数据，如版本信息、描述等。

**示例：**

```yaml
metadata:
  name: my-deployment
  namespace: default
  labels:
    app: my-app
```

---

### **4. spec**

- **作用**：定义 Kubernetes 资源的期望状态（即声明式配置）。这个字段是根据具体资源类型来定义的，比如 Deployment 中的 `spec` 会包括 `replicas`（副本数量）、`selector`（选择器）和 Pod 模板；而 Service 的 `spec` 会包括暴露的端口、选择器等。
- **结构**：每种资源的 `spec` 字段都不相同，具体内容取决于资源类型。
    - 对于 `Pod`，`spec` 中定义容器的配置。
    - 对于 `Deployment`，`spec` 中定义副本数、Pod 模板等。
    - 对于 `Service`，`spec` 中定义暴露端口、服务类型等。

**示例**：

```yaml
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```

---

## **完整示例**

以下是一个完整的 `Deployment` YAML 文件示例，展示了四个顶级字段：

```yaml
apiVersion: apps/v1  # 定义 API 版本
kind: Deployment     # 资源类型是 Deployment
metadata:            # 资源的元数据
  name: my-deployment
  labels:
    app: my-app
spec:                # 资源的期望状态配置
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```

### **解释：**

- **apiVersion**：指定 Deployment 属于 `apps/v1` API 组。
- **kind**：指定资源类型为 `Deployment`。
- **metadata**：为该 Deployment 赋予了名称 `my-deployment` 并添加了标签 `app: my-app`。
- **spec**：
    - **replicas**：指定希望运行的 Pod 副本数为 3。
    - **selector**：定义标签选择器，选择所有具有标签 `app: my-app` 的 Pod。
    - **template**：定义 Pod 模板，包括 Pod 的标签 `app: my-app` 和一个运行 `nginx` 容器的配置。

---

## **总结**

Kubernetes 中定义资源时，YAML 文件的基本结构始终包含以下四个顶级字段：

1. **`apiVersion`**：指定 API 版本，定义资源所属的 API 组。
2. **`kind`**：资源类型，如 Pod、Service、Deployment 等。
3. **`metadata`**：元数据，包含资源的名称、标签等信息。
4. **`spec`**：定义资源的期望状态，根据具体的资源类型不同而变化。

这些字段为 Kubernetes 提供了一个声明式管理系统，确保用户只需定义期望状态，Kubernetes 会自动调整系统以匹配这些声明的配置。


## create pod
以下是一个使用 Nginx 镜像创建的 Pod 的 YAML 文件示例。在这个示例中，我们定义了一个简单的 Pod，使用官方的 `nginx` 镜像，运行 Nginx Web 服务器，并暴露端口 80。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.21.6   # 使用 Nginx 官方镜像
    ports:
    - containerPort: 80   # 暴露 Nginx 服务器的 80 端口
```

### **说明：**

1. **apiVersion**: `v1` 是创建 Pod 的 API 版本。
2. **kind**: `Pod` 表示资源类型是 Pod。
3. **metadata**:
    - **name**: `nginx-pod` 是这个 Pod 的名称。
    - **labels**: 使用 `app: nginx` 标签来标记该 Pod（标签是可选的，可以用来组织和选择资源）。
4. **spec**:
    - **containers**: 定义了该 Pod 运行的容器列表。
        - **name**: 容器名称为 `nginx`。
        - **image**: 使用 Nginx 官方镜像 `nginx:1.21.6`。
        - **ports**: 暴露容器的 `80` 端口。

### **部署步骤：**

1. 将上述 YAML 文件保存为 `nginx-pod.yaml`。

2. 在终端中，使用 `kubectl` 命令将该 Pod 部署到 Kubernetes 集群中：

   ```bash
   kubectl apply -f nginx-pod.yaml
   ```

3. 使用以下命令查看 Pod 的状态，确保它运行正常：

   ```bash
   kubectl get pods
   ```

4. 您还可以通过以下命令查看 Pod 的详细信息：

   ```bash
   kubectl describe pod nginx-pod
   ```

5. 如果想进入到 Pod 内部，可以使用 `kubectl exec` 命令：

   ```bash
   kubectl exec -it nginx-pod -- /bin/bash
   ```

通过上述步骤，您将能够创建并运行一个 Nginx Web 服务器 Pod。