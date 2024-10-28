# Kubernetes - PODs
## Step-01: PODs Introduction
在 Kubernetes（常简称为 K8s）中，**Pod** 是部署和管理应用程序的最小单元。它可以被视为一组容器的抽象，这些容器共享相同的网络和存储，并共同作为一个整体运行。

### **什么是 Pod？**

#### **Pod 的概念**

- **基本单位**：Pod 是 Kubernetes 中可以创建、调度和管理的最小单元。它封装了一个或多个容器，这些容器被视为一个单元来运行和管理。
- **共享资源**：Pod 内的容器共享网络命名空间（IP 地址和端口空间）和存储卷。这意味着它们可以通过 `localhost` 进行通信，并可以访问相同的持久化存储。
- **生命周期**：Pod 有自己的生命周期，从创建、运行到终止。Pod 内的所有容器共享这个生命周期。

#### **Pod 的特性**

1. **共享网络**：所有容器共享同一个网络栈，可以直接通过 `localhost` 访问彼此的端口。
2. **共享存储**：可以将一个或多个卷挂载到 Pod，所有容器都可以访问这些卷。
3. **原子性调度**：Pod 作为一个整体被调度到节点上，确保其内的容器一起运行。

#### **Pod 的用途**

- **单容器应用**：最常见的是在一个 Pod 中运行一个容器，代表一个微服务或应用程序实例。
- **多容器协作**：在需要紧密协作的场景下，将多个容器放入一个 Pod，以共享资源和环境。

### **什么是多容器 Pod？**

#### **多容器 Pod 的概念**

- **定义**：多容器 Pod 是指在同一个 Pod 内运行多个容器。这些容器紧密协作，共同完成某项功能或服务。
- **协同工作**：容器之间可以通过共享的网络和存储进行高效通信，实现功能互补。

#### **多容器 Pod 的模式**

1. **Sidecar 模式**：

    - **描述**：辅助主容器，提供增强或支持功能。
    - **示例**：日志收集器、安全代理、配置更新器。

2. **Ambassador 模式**：

    - **描述**：充当主容器和外部世界之间的代理，帮助处理网络通信。
    - **示例**：反向代理、负载均衡器。

3. **Adapter 模式**：

    - **描述**：将主容器的输出转换为另一种格式或接口。
    - **示例**：日志格式转换器、协议适配器。

#### **多容器 Pod 的用途**

- **日志和监控**：一个容器运行应用，另一个容器负责日志收集和转发。
- **代理和网关**：在主应用前添加一个代理容器，处理安全、路由等事务。
- **数据处理流水线**：多个容器按顺序处理数据，每个容器负责不同的处理阶段。

### **为什么使用多容器 Pod？**

- **资源共享**：当多个容器需要共享相同的存储或网络资源时，将它们放在同一个 Pod 中更为便利。
- **模块化设计**：将不同的功能拆分为独立的容器，提升代码的可维护性和复用性。
- **性能优化**：容器之间通过共享内存和本地通信，提高数据交换的效率。

### **需要注意的事项**

- **生命周期管理**：Pod 内的容器共享生命周期，需确保容器的启动顺序和依赖关系正确。
- **资源配置**：为每个容器设置适当的资源限制（CPU、内存），防止资源争夺。
- **故障隔离**：容器之间的故障可能会相互影响，需要设计健壮的错误处理机制。
- **健康检查**：为每个容器配置健康检查，确保应用的可用性。

### **总结**

- **Pod 是 Kubernetes 中的核心概念**，用于封装一个或多个紧密协作的容器。
- **多容器 Pod** 允许在一个部署单元中运行多个容器，提供更大的灵活性和功能组合。
- **正确使用 Pod 和多容器模式**，可以构建高效、可伸缩和易于管理的容器化应用程序。





## Step-02: PODs Demo

### Get Worker Nodes Status
要验证 Kubernetes 工作节点（worker nodes）是否就绪，您可以使用 `kubectl get nodes` 命令。该命令将列出集群中的所有节点及其状态。

**以下是具体步骤：**

1. **打开终端**，确保您有权限访问 Kubernetes 集群。

2. **运行以下命令：**

   ```bash
   kubectl get nodes
   ```

3. **检查输出结果：**

   该命令的输出将显示所有节点及其当前状态。请关注 `STATUS` 列；节点就绪时会显示为 `Ready`。

   **示例输出：**

   ```bash
   NAME           STATUS   ROLES           AGE     VERSION
   master-node    Ready    control-plane   10d     v1.21.0
   worker-node1   Ready    <none>          10d     v1.21.0
   worker-node2   Ready    <none>          10d     v1.21.0
   ```

   - 如果某个节点显示为 `NotReady`，表示该节点存在需要处理的问题。

4. **（可选）查看特定节点的详细信息：**

   如果您需要了解某个节点的更多细节，可以运行：

   ```bash
   kubectl describe node <节点名称>
   ```

   将 `<节点名称>` 替换为您要检查的节点名。此命令将提供该节点的详细信息，包括条件、资源和相关事件。

**附加提示：**

- **查看节点的更多信息：**

  您可以运行以下命令以获取更详细的信息，包括节点的内部和外部 IP 地址：

  ```bash
  kubectl get nodes -o wide
  ```

- **验证所有 Pod 的运行状态：**

  节点的问题有时会反映在 Pod 的状态上。您可以检查所有命名空间中的所有 Pod：

  ```bash
  kubectl get pods --all-namespaces
  ```

- **监控节点资源：**

  确保节点具有足够的 CPU、内存和磁盘空间。资源耗尽可能导致节点状态变为 `NotReady`。

### Create a Pod

### List Pods

### List Pods with wide option

### What happened in the backgroup when above command is run?

### Describe Pod

### Access Application


### Delete Pod


## Step-03: NodePort Service Introduction

### What are Services in k8s?

### What is a NodePort Service?

### How it works?


## Step-04: Demo - Expose Pod with a Service


## Step-05: Interact with a Pod

### Verify Pod Logs


### Connect to Container in a POD

## Step-06: Get YAML Output of Pod & Service

### Get YAML Output

### Step-07: Clean-Up
