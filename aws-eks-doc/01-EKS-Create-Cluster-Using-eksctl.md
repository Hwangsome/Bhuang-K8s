# eksctl
### 什么是 `eksctl`？

`eksctl` 是一个简化和管理 **AWS Elastic Kubernetes Service (EKS)** 的命令行工具，它由 **Weaveworks** 开发。通过 `eksctl`，您可以快速创建、配置和管理 EKS 集群，而不需要手动配置复杂的 YAML 文件或通过 AWS 控制台逐步设置。

`eksctl` 使用 AWS 的 SDK 和 CloudFormation 模板，将复杂的 EKS 集群操作抽象化为简单的命令行操作，非常适合开发人员和运维团队。

---

### `eksctl` 的主要功能

1. **创建 EKS 集群**  
   您可以在几分钟内使用简单命令创建一个功能齐全的 EKS 集群，包括 VPC、子网、节点组等。

2. **管理节点组**  
   支持管理 Amazon EC2 和 Fargate 节点组，包括添加/删除节点组、调整节点规模等。

3. **配置集群**  
   可以配置集群的 Kubernetes 版本、IAM 角色、网络策略等。

4. **删除集群**  
   一键删除所有与集群相关的资源，包括 VPC、节点等。

5. **支持 YAML 配置文件**  
   可以通过 YAML 文件定义集群的配置并使用 `eksctl` 命令直接部署。

---

### 如何安装 `eksctl`

#### 1. **通过 `brew` 安装（macOS 和 Linux）**

```bash
brew tap weaveworks/tap
brew install weaveworks/tap/eksctl
```

#### 2. **通过 `curl` 安装**

```bash
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
```

#### 3. **验证安装**

```bash
eksctl version
```

输出示例：
```plaintext
0.157.0
```

---

### 快速入门示例

#### 1. **创建一个 EKS 集群**

```bash
eksctl create cluster --name my-cluster --region us-east-1 --nodes 3 --node-type t3.medium
```

解释：
- `--name`：指定集群名称。
- `--region`：指定 AWS 区域。
- `--nodes`：指定节点数量。
- `--node-type`：指定节点实例类型。

> 默认情况下，它会创建一个新的 VPC 和子网。如果已有 VPC，可通过配置文件指定。

#### 2. **使用配置文件创建集群**

创建一个 YAML 文件 `cluster.yaml`：

```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: my-cluster
  region: us-east-1

nodeGroups:
  - name: ng-1
    instanceType: t3.medium
    desiredCapacity: 2
    minSize: 1
    maxSize: 3
```

运行命令：

```bash
eksctl create cluster -f cluster.yaml
```

#### 3. **列出所有 EKS 集群**

```bash
eksctl get cluster
```

#### 4. **添加新节点组**

```bash
eksctl create nodegroup --cluster my-cluster --name ng-2 --nodes 2 --node-type t3.large
```

#### 5. **删除集群**

```bash
eksctl delete cluster --name my-cluster
```

---

### 高级用法

#### 1. **与 `kubectl` 集成**
`eksctl` 创建的 EKS 集群会自动配置 `kubeconfig` 文件，因此可以直接使用 `kubectl` 操作集群：

```bash
kubectl get nodes
kubectl apply -f deployment.yaml
```

#### 2. **与 Fargate 集成**
在 EKS 中启用 Fargate：

```bash
eksctl create cluster --name my-fargate-cluster --region us-west-2 --fargate
```

#### 3. **升级集群 Kubernetes 版本**

```bash
eksctl upgrade cluster --name my-cluster --region us-east-1 --approve
```

#### 4. **升级节点组**

```bash
eksctl upgrade nodegroup --cluster my-cluster --name ng-1 --region us-east-1
```

---

### `eksctl` 的优缺点

#### 优点
1. **快速简单**：只需几条命令即可完成复杂的操作。
2. **配置灵活**：支持 YAML 文件定义和直接 CLI 命令。
3. **自动化**：默认集成了最佳实践（如安全组、IAM 角色等）。
4. **与 AWS 服务无缝集成**：支持 EC2、Fargate 等。

#### 缺点
1. **依赖 AWS CloudFormation**：如果 CloudFormation 堆栈出现问题，可能会影响集群管理。
2. **功能有限**：复杂的自定义需求可能需要结合其他工具（如 Terraform）。

---

### 总结

`eksctl` 是一个功能强大、简单易用的工具，适合用于快速创建和管理 EKS 集群，特别是在开发和测试环境中。如果您需要更高级的部署或跨云服务集成，可以考虑与其他基础设施即代码（IaC）工具如 Terraform 结合使用。

# eks cluster
## 介绍
### 什么是 AWS EKS Cluster？

AWS EKS Cluster（Amazon Elastic Kubernetes Service 集群）是一个托管的 **Kubernetes 集群**，由 AWS 提供。通过 EKS，您可以运行、扩展和管理 Kubernetes 应用程序，而无需自行设置 Kubernetes 控制平面（Control Plane）或节点（Worker Nodes）。EKS 集群结合了 Kubernetes 的开放性和 AWS 的高可用性、安全性以及与 AWS 服务的深度集成。

---

### EKS Cluster 的基本架构

一个 EKS 集群包括以下关键组件：

#### 1. **控制平面 (Control Plane)**
- **AWS 托管**：AWS 管理 Kubernetes 控制平面组件（如 API Server、etcd 和调度器），这些组件运行在多个可用区中以实现高可用性。
- **职责**：负责 Kubernetes 的核心功能，如 API 处理、调度和状态管理。

#### 2. **节点 (Worker Nodes)**
- **EC2 节点**：运行实际的容器化工作负载（Pod）。
- **Fargate 节点**：无服务器选项，AWS 直接管理底层计算资源。
- **职责**：运行 Pod、处理计算任务。

#### 3. **VPC（虚拟私有云）和网络**
- EKS 集群需要在 VPC 内运行，您可以使用 AWS 提供的 VPC 或自定义 VPC。
- Kubernetes 服务通过 **AWS Load Balancer**（如 ALB 和 NLB）暴露给外部。

#### 4. **IAM 集成**
- 控制权限访问（通过 AWS IAM 控制用户和服务对集群的访问权限）。
- 节点和 Pod 使用 IAM 角色访问 AWS 服务。

---

### EKS Cluster 的核心功能

1. **完全托管的 Kubernetes**
   - AWS 管理 Kubernetes 控制平面，包括故障恢复和软件升级。
   - 提供与开源 Kubernetes 100% 兼容的体验。

2. **多可用区高可用性**
   - 控制平面自动在多个可用区中复制。
   - 支持高可用性和灾难恢复。

3. **无服务器选项**
   - 使用 AWS Fargate，无需管理底层计算资源。

4. **自动化的节点组管理**
   - 自动扩展节点组（通过集成的 Cluster Autoscaler 或 Amazon EC2 Auto Scaling）。
   - 节点生命周期管理，如自动修复和更新。

5. **与 AWS 服务集成**
   - 原生支持 AWS 服务（如 IAM、CloudWatch、VPC、ALB）。
   - 通过 IAM Roles for Service Accounts (IRSA) 控制 Kubernetes 工作负载访问 AWS 资源。

6. **安全和合规性**
   - 使用 VPC 隔离网络流量。
   - 支持基于 Kubernetes RBAC 和 AWS IAM 的访问控制。

---

### 如何创建 EKS Cluster

#### 方法 1：通过 `eksctl` 工具
以下示例使用 `eksctl` 快速创建一个 EKS 集群：

```bash
eksctl create cluster --name my-cluster --region us-east-1 --nodes 3 --node-type t3.medium
```

#### 方法 2：通过 AWS 管理控制台
1. 登录到 AWS 管理控制台。
2. 导航到 **EKS** 服务。
3. 点击 “Create Cluster”，配置以下信息：
   - **Cluster name**: 输入集群名称。
   - **Kubernetes version**: 选择所需版本（例如 `1.27`）。
   - **Cluster Service Role**: 分配 EKS 所需的 IAM 角色。
4. 配置网络（VPC 和子网），然后点击 “Create”。

#### 方法 3：通过 AWS CLI
如果您更喜欢命令行，可以使用 AWS CLI 和 CloudFormation 模板。

示例命令：
```bash
aws eks create-cluster \
    --name my-cluster \
    --role-arn arn:aws:iam::<account-id>:role/eks-role \
    --resources-vpc-config subnetIds=subnet-xyz123,securityGroupIds=sg-abc456
```

---

### 配置文件示例（通过 `eksctl` 创建集群）

您可以使用 YAML 文件定义 EKS 集群：

```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: my-cluster
  region: us-east-1

nodeGroups:
  - name: standard-workers
    instanceType: t3.medium
    desiredCapacity: 3
    minSize: 2
    maxSize: 5
    iam:
      withAddonPolicies:
        autoScaler: true
```

使用以下命令创建：
```bash
eksctl create cluster -f cluster-config.yaml
```

---

### 管理 EKS 集群

#### 1. **查看集群列表**
```bash
eksctl get cluster
```

#### 2. **查看节点状态**
```bash
kubectl get nodes
```

#### 3. **升级 Kubernetes 版本**
```bash
eksctl upgrade cluster --name my-cluster --region us-east-1 --approve
```

#### 4. **添加节点组**
```bash
eksctl create nodegroup --cluster my-cluster --name extra-nodes --nodes 2 --node-type t3.large
```

#### 5. **删除集群**
```bash
eksctl delete cluster --name my-cluster
```

---

### 常见的 EKS 用例

1. **微服务架构**
   - 在 Kubernetes 上运行微服务应用程序。
   - 使用 Kubernetes Ingress 和 AWS ALB/NLB 处理流量。

2. **大规模数据处理**
   - 使用 EKS 管理大规模数据处理工作流（如使用 Spark 或 TensorFlow）。

3. **CI/CD 管道**
   - 与 GitLab、Jenkins 等集成，提供基于 Kubernetes 的 CI/CD 环境。

4. **机器学习**
   - 在 EKS 中运行机器学习工作负载，使用 AWS 的 GPU 实例加速训练。

---

### EKS 集群的优缺点

#### 优点
1. **托管服务降低运维复杂性**
   - AWS 自动管理 Kubernetes 控制平面，使您可以专注于工作负载。

2. **高可用性**
   - 自动在多个可用区中复制控制平面，确保稳定性。

3. **与 AWS 服务的深度集成**
   - 原生支持 IAM、CloudWatch、ALB 等。

4. **可扩展性**
   - 自动扩展节点组，支持高峰工作负载。

5. **安全性**
   - 使用 IAM 和 RBAC 控制访问权限。

#### 缺点
1. **价格昂贵**
   - 控制平面有固定费用（$0.10/小时），加上节点的 EC2 实例费用。
2. **复杂性**
   - 初学者可能会觉得网络配置（如 VPC、子网、路由表）复杂。
3. **Kubernetes 升级可能需要手动干预**
   - 尽管 EKS 提供升级选项，但可能需要手动协调节点和应用程序的升级。

---

### 总结

EKS 是一种强大而灵活的托管 Kubernetes 服务，适合需要高可用性和与 AWS 服务无缝集成的生产环境。通过工具如 `eksctl` 和 AWS CLI，可以简化 EKS 集群的创建和管理。如果您需要快速部署容器化工作负载或构建可扩展的微服务架构，EKS 是一个强有力的选择。

## Fargate profile
### EKS Cluster 基本架构中的 Fargate Profile

在 **Amazon EKS** 中，除了 EC2 节点组，**Fargate Profile** 是另一个重要的组成部分，它为 Kubernetes 提供了 **无服务器计算模式**。Fargate Profile 是 EKS 集群中管理 Fargate 工作负载的配置文件，定义了哪些 Pod 应该运行在 Fargate 上。

---

### 什么是 Fargate Profile？

Fargate Profile 是一个配置对象，它允许您为 Kubernetes 的 **Pod** 指定自动运行在 AWS Fargate 上的策略，而不需要手动管理底层计算资源（例如 EC2 实例）。

- **Pod 的无服务器计算选项**：Fargate Profile 使得运行 Pod 无需关心底层的节点。
- **按需计费**：基于运行 Pod 所需的 vCPU 和内存计费。
- **独立于节点组**：Pod 直接分配到 Fargate，不与 EC2 节点组混合。

---

### Fargate Profile 的架构特性

1. **选择性调度**
   - 您可以通过 Fargate Profile 指定基于命名空间或标签选择器（label selector）的调度规则，告诉 Kubernetes 哪些 Pod 应该运行在 Fargate 上。

2. **自动化基础设施管理**
   - AWS 自动为运行在 Fargate 上的 Pod 配置底层计算、网络、安全组等，用户无需管理。

3. **深度网络集成**
   - 每个 Fargate Pod 都在 VPC 中直接分配一个 ENI（弹性网络接口），拥有独立的网络隔离。

4. **与 IAM 集成**
   - 使用 IAM Roles for Service Accounts（IRSA），可以为运行在 Fargate 上的 Pod 分配特定的 AWS 资源访问权限。

---

### Fargate Profile 的核心组件

1. **命名空间**
   - Fargate Profile 通过 Kubernetes 命名空间来过滤哪些 Pod 应该使用 Fargate。例如，所有属于 `namespace: fargate-ns` 的 Pod 都会运行在 Fargate 上。

2. **标签选择器**
   - 配置 Pod 的 `Label Selector` 来精确选择需要运行在 Fargate 上的 Pod。

3. **子网**
   - 您需要指定 Fargate Profile 使用的 VPC 子网，AWS 会为 Fargate Pod 分配 IP 地址。

---

### 创建 Fargate Profile

#### 1. **通过 AWS 控制台**
- 在 EKS 控制台的 "Compute" 下选择 "Add Fargate Profile"。
- 配置以下内容：
   - **Profile name**: Fargate 配置文件的名称。
   - **Namespace**: 指定 Kubernetes 命名空间。
   - **Subnets**: 选择 VPC 的子网。
   - **Tags**: 可选，添加标签。

#### 2. **通过 `eksctl`**

以下命令创建一个 Fargate Profile，为 `fargate-ns` 命名空间的 Pod 提供 Fargate 运行时支持：

```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: my-cluster
  region: us-east-1

fargateProfiles:
  - name: fargate-profile
    selectors:
      - namespace: fargate-ns
    subnets:
      - subnet-0abcdef1234567890
      - subnet-0abcdef1234567891
```

运行命令：
```bash
eksctl create cluster -f cluster-config.yaml
```

#### 3. **通过 AWS CLI**
使用 AWS CLI 配置 Fargate Profile：

```bash
aws eks create-fargate-profile \
    --cluster-name my-cluster \
    --fargate-profile-name my-fargate-profile \
    --pod-execution-role-arn arn:aws:iam::123456789012:role/AmazonEKSFargatePodExecutionRole \
    --selectors namespace=fargate-ns \
    --subnets subnet-0abcdef1234567890 subnet-0abcdef1234567891
```

---

### 使用 Fargate Profile 调度 Pod

创建一个运行在 Fargate 上的 Pod 示例：

1. 创建命名空间 `fargate-ns`：
   ```bash
   kubectl create namespace fargate-ns
   ```

2. 部署示例应用：
   创建 `nginx-deployment.yaml`：
   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: nginx
     namespace: fargate-ns
   spec:
     replicas: 2
     selector:
       matchLabels:
         app: nginx
     template:
       metadata:
         labels:
           app: nginx
       spec:
         containers:
         - name: nginx
           image: nginx:latest
           ports:
           - containerPort: 80
   ```

   应用部署：
   ```bash
   kubectl apply -f nginx-deployment.yaml
   ```

3. 验证 Pod 是否运行在 Fargate 上：
   ```bash
   kubectl get pods -n fargate-ns -o wide
   ```

   您会看到每个 Pod 分配了独立的 IP 地址，而不是运行在节点组上的节点。

---

### Fargate Profile 的优点

1. **无服务器**
   - 无需管理底层 EC2 节点的生命周期或配置，AWS 自动分配计算资源。

2. **成本优化**
   - 按需付费，仅为运行的 Pod 使用的 vCPU 和内存计费。

3. **安全性**
   - 每个 Pod 在独立的 ENI 中运行，提供网络隔离。

4. **弹性**
   - 无需预置节点，Pod 启动时资源按需分配。

5. **与 AWS 服务的集成**
   - 支持 IRSA，为 Pod 提供 AWS 服务的精细权限控制。

---

### Fargate Profile 的局限性

1. **性能限制**
   - Fargate 适合运行轻量级任务，但对计算密集型任务可能效率较低。

2. **资源配额**
   - 每个 Fargate Pod 的最大 vCPU 和内存有固定配额（如最大 4 vCPU 和 30GB 内存）。

3. **调试复杂**
   - 由于无服务器特性，调试底层资源的能力有限。

4. **网络成本**
   - 每个 Pod 使用独立的 ENI，可能增加网络相关费用。

---

### 适用场景

1. **开发和测试环境**
   - 快速部署小型 Kubernetes 应用，无需设置节点组。

2. **轻量级服务**
   - 运行无状态的 Web 服务或 API 网关。

3. **事件驱动任务**
   - 运行短生命周期的事件驱动任务或批处理作业。

4. **多租户环境**
   - 提供强隔离性，支持多租户 Kubernetes 应用。

---

### 总结

**Fargate Profile** 是 EKS 集群架构中一个重要的无服务器计算选项，适合需要弹性、隔离性且对底层计算资源无管理需求的 Kubernetes 工作负载。通过配置 Fargate Profile，您可以灵活选择哪些 Pod 应该运行在 Fargate 上，从而降低运维负担并优化成本。如果有复杂的计算需求或需要精细化控制，则可以结合 EC2 节点组使用。

## eks cluster 是怎么工作的
![img.png](img%2Fimg.png)


##  Step01: Create EKS Cluster using eksctl
- note: 这里可能要花费5~20分钟的时间来创建集群
```shell
# Create Cluster
eksctl create cluster --name=eksdemo1 \
                      --region=us-east-1 \
                      --zones=us-east-1a,us-east-1b \
                      --without-nodegroup 

# Get List of clusters
eksctl get cluster                  
```

## Step02- Create & Associate IAM OIDC Provider for our EKS Cluster
### 如何为 EKS 集群创建并关联 IAM OIDC 提供商

在 **Amazon EKS** 中，使用 **IAM OIDC 提供商** 可以让 Kubernetes 中的服务账户（Service Account）直接使用 IAM 角色，从而实现对 AWS 资源的细粒度访问控制。

以下是详细步骤：

---

### 第一步：验证 EKS 集群是否已关联 OIDC 提供商

在创建新的 OIDC 提供商之前，检查当前 EKS 集群是否已经关联了 OIDC 提供商。

1. 使用 AWS CLI 检查集群的 OIDC 提供商：
   ```bash
   aws eks describe-cluster --name <集群名称> --query "cluster.identity.oidc.issuer" --output text
   ```

   示例输出：
   ```
   https://oidc.eks.<region>.amazonaws.com/id/<eks-cluster-id>
   ```

2. 如果您看到一个类似这样的 URL，这说明集群已关联 OIDC 提供商。如果没有返回任何 URL，请继续执行以下步骤创建 OIDC 提供商。

---

### 第二步：为集群创建并关联 OIDC 提供商

#### 选项 1：使用 AWS CLI 创建 OIDC 提供商

1. **获取集群的 OIDC 提供商 URL**：
   ```bash
   aws eks describe-cluster --name <集群名称> --query "cluster.identity.oidc.issuer" --output text
   ```

2. **创建 OIDC 提供商**：

   使用 AWS CLI 命令创建 OIDC 提供商：
   ```bash
   aws iam create-open-id-connect-provider \
       --url $(aws eks describe-cluster --name <集群名称> --query "cluster.identity.oidc.issuer" --output text) \
       --client-id-list sts.amazonaws.com \
       --thumbprint-list <thumbprint>
   ```

   - `--url` 是集群的 OIDC 提供商 URL。
   - `--client-id-list` 设置为 `sts.amazonaws.com`，表示允许 STS 使用 OIDC 提供商。
   - `--thumbprint-list` 是 OIDC 提供商的证书指纹（thumbprint）。

   **如何获取指纹？**
   - 使用 `openssl` 从 OIDC 提供商的 HTTPS 证书中提取指纹：
     ```bash
     openssl s_client -connect <oidc-url>:443 -showcerts </dev/null 2>/dev/null | openssl x509 -fingerprint -noout
     ```

#### 选项 2(prefer, 我使用的是这种方式)：使用 `eksctl` 自动化创建

如果您使用 `eksctl`，可以直接通过以下命令创建并关联 OIDC 提供商：

```bash
eksctl utils associate-iam-oidc-provider --cluster <集群名称> --region <region名字> --approve
```

此命令会自动完成关联操作，无需手动提取指纹。

```shell
eksctl utils associate-iam-oidc-provider \
    --region us-east-1 \
    --cluster eksdemo1 \
    --approve
```
![img_1.png](img%2Fimg_1.png)

---

### 第三步：验证 OIDC 提供商的关联

1. 使用 AWS CLI 列出所有 IAM OIDC 提供商：
   ```bash
   aws iam list-open-id-connect-providers
   ```

2. 检查输出中是否包含类似以下内容：
   ```
   arn:aws:iam::<account-id>:oidc-provider/oidc.eks.<region>.amazonaws.com/id/<eks-cluster-id>
   ```

3. 您也可以通过 AWS 管理控制台中的 “IAM -> OIDC Providers” 页面查看。

---

### 第四步：测试 OIDC 提供商

关联 OIDC 提供商后，您可以为 Kubernetes 的服务账户配置 IAM 角色进行测试。

#### 1. 创建 IAM 信任策略

创建一个名为 `trust-policy.json` 的文件，内容如下：

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<account-id>:oidc-provider/oidc.eks.<region>.amazonaws.com/id/<eks-cluster-id>"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.<region>.amazonaws.com/id/<eks-cluster-id>:sub": "system:serviceaccount:<namespace>:<service-account-name>"
        }
      }
    }
  ]
}
```

- 替换 `<account-id>`、`<region>` 和 `<eks-cluster-id>` 为您的账户和集群信息。
- 替换 `<namespace>` 和 `<service-account-name>` 为 Kubernetes 服务账户的命名空间和名称。

#### 2. 创建 IAM 角色

运行以下命令创建角色：
```bash
aws iam create-role \
    --role-name EKS-Test-Role \
    --assume-role-policy-document file://trust-policy.json
```

#### 3. 附加 IAM 权限

为角色附加所需的权限，例如 `AmazonS3ReadOnlyAccess`：
```bash
aws iam attach-role-policy --role-name EKS-Test-Role --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
```

#### 4. 将 IAM 角色关联到 Kubernetes 服务账户

通过 `eksctl` 命令将 IAM 角色关联到服务账户：

```bash
eksctl create iamserviceaccount \
    --name <service-account-name> \
    --namespace <namespace> \
    --cluster <cluster-name> \
    --attach-role-arn arn:aws:iam::<account-id>:role/EKS-Test-Role \
    --approve
```

验证服务账户是否已关联：
```bash
kubectl get serviceaccount <service-account-name> -n <namespace> -o yaml
```

---

### 总结

通过上述步骤，您可以成功为 EKS 集群创建并关联 IAM OIDC 提供商，并通过 Kubernetes 的服务账户使用 IAM 角色访问 AWS 资源。

**流程概览**：
1. 验证是否已关联 OIDC 提供商。
2. 创建并关联 IAM OIDC 提供商。
3. 创建测试用的 IAM 角色和信任策略。
4. 将 IAM 角色关联到 Kubernetes 服务账户。
5. 验证设置是否生效。

## Step03-Create EC2 Keypair
- Create a new EC2 Keypair with name as kube-demo 
- This keypair we will use it when creating the EKS NodeGroup. 
- This will help us to login to the EKS Worker Nodes using Terminal.


### 如何创建 EC2 Key Pair

**EC2 Key Pair** 是 AWS 用来安全连接 EC2 实例的 SSH 密钥对，由 **私钥 (private key)** 和 **公钥 (public key)** 组成。AWS 保存公钥并将其与 EC2 实例关联，而私钥则由用户自行保存，用于通过 SSH 连接 EC2 实例。

---

### ：使用 AWS CLI 创建 Key Pair

1. **运行以下命令**以创建新的 Key Pair：
   ```bash
   aws ec2 create-key-pair --key-name <KeyName> --query 'KeyMaterial' --output text > <KeyName>.pem
   ```

   参数说明：
   - `--key-name <KeyName>`：替换为您希望的密钥对名称（例如 `my-key`）。
   - `--query 'KeyMaterial'`：提取密钥内容。
   - `--output text`：输出密钥内容为纯文本格式。
   - `> <KeyName>.pem`：将密钥内容保存到文件（如 `my-key.pem`）。

2. **检查私钥文件是否生成**：
   ```bash
   ls -l <KeyName>.pem
   ```

3. **修改私钥文件权限**：
   ```bash
   chmod 400 <KeyName>.pem
   ```

   设置权限为只读，以防止其他用户访问。

---

### 使用 Key Pair 连接 EC2 实例

1. 启动 EC2 实例时，选择关联的 Key Pair。
2. 使用 SSH 连接实例：
   ```bash
   ssh -i <KeyName>.pem ec2-user@<InstancePublicIP>
   ```

---

### 注意事项

1. **私钥的安全性**：
   - 私钥文件只会在创建时提供一次，请妥善保存。
   - 不要将私钥文件泄露或上传到公共存储（如 GitHub）。
   - 使用 `chmod 400` 限制文件访问权限。

2. **丢失私钥的解决办法**：
   - 如果丢失了私钥，您无法直接访问使用该 Key Pair 的实例。
   - 解决办法：
      - 创建一个新的 Key Pair。
      - 使用 AWS Systems Manager 的 Session Manager 登录实例。
      - 或通过实例的用户数据脚本注入新的公钥。

3. **Key Pair 限制**：
   - 一个 Key Pair 可以关联多个实例。
   - AWS 不会存储私钥，确保在创建时立即下载。

---

## Step04- Create Node Group with additional Add-Ons in Public Subnets

--- 

- These add-ons will create the respective IAM policies for us automatically within our Node Group role.
```shell
# Create Public Node Group   
eksctl create nodegroup --cluster=eksdemo1 \
                       --region=us-east-1 \
                       --name=eksdemo1-ng-public1 \
                       --node-type=t3.medium \
                       --nodes=2 \
                       --nodes-min=2 \
                       --nodes-max=4 \
                       --node-volume-size=20 \
                       --ssh-access \
                       --ssh-public-key=kube-demo \
                       --managed \
                       --asg-access \
                       --external-dns-access \
                       --full-ecr-access \
                       --appmesh-access \
                       --alb-ingress-access 
```

要使用 `eksctl` 创建一个带有附加组件的 **Node Group**（在公共子网中）并自动为 Node Group 的角色创建相应的 IAM 策略，可以按照以下步骤操作。您的命令已正确地包含了这些附加参数，这些参数会自动为我们创建所需的 IAM 策略。

---

### 1. 创建 Node Group 的完整命令解释

```bash
eksctl create nodegroup \
  --cluster=eksdemo1 \                     # 集群名称
  --region=us-east-1 \                     # AWS 区域
  --name=eksdemo1-ng-public1 \             # Node Group 名称
  --node-type=t3.medium \                  # 节点实例类型
  --nodes=2 \                              # 初始节点数量
  --nodes-min=2 \                          # 最小节点数量
  --nodes-max=4 \                          # 最大节点数量
  --node-volume-size=20 \                  # 每个节点的 EBS 卷大小 (GiB)
  --ssh-access \                           # 启用 SSH 访问
  --ssh-public-key=kube-demo \             # 指定 SSH 公钥
  --managed \                              # 使用 AWS 托管节点组
  --asg-access \                           # 启用 Auto Scaling 组权限
  --external-dns-access \                  # 添加 External DNS 的 IAM 权限
  --full-ecr-access \                      # 添加对 Amazon ECR 的完整访问权限
  --appmesh-access \                       # 添加对 App Mesh 的权限
  --alb-ingress-access                     # 添加对 ALB Ingress Controller 的权限
```

---

### 2. 每个选项的作用

1. **`--cluster`**: 指定目标 EKS 集群名称。
2. **`--region`**: 指定 AWS 区域。
3. **`--name`**: 设置 Node Group 的名称。
4. **`--node-type`**: 指定 EC2 实例类型。
5. **`--nodes`, `--nodes-min`, `--nodes-max`**: 设置节点组的初始、最小和最大节点数。
6. **`--node-volume-size`**: 配置每个节点的 EBS 卷大小。
7. **`--ssh-access` 和 `--ssh-public-key`**: 启用对节点的 SSH 访问，并指定公钥。
8. **`--managed`**: 创建 AWS 托管的节点组。
9. **`--asg-access`**: 为 Auto Scaling 组提供 IAM 权限，用于扩展和缩减节点。
10. **`--external-dns-access`**: 为 External DNS 添加必要的 IAM 权限。
11. **`--full-ecr-access`**: 添加对 Amazon ECR 的完整权限（用于拉取容器镜像）。
12. **`--appmesh-access`**: 添加对 App Mesh 的权限（用于服务网格）。
13. **`--alb-ingress-access`**: 添加对 ALB Ingress Controller 的权限。

---

### 3. 执行命令

运行完整的命令：

```bash
eksctl create nodegroup \
  --cluster=eksdemo1 \
  --region=us-east-1 \
  --name=eksdemo1-ng-public1 \
  --node-type=t3.medium \
  --nodes=2 \
  --nodes-min=2 \
  --nodes-max=4 \
  --node-volume-size=20 \
  --ssh-access \
  --ssh-public-key=kube-demo \
  --managed \
  --asg-access \
  --external-dns-access \
  --full-ecr-access \
  --appmesh-access \
  --alb-ingress-access
```

---

### 4. 验证 Node Group 和附加组件

#### 验证 Node Group 是否已成功创建：
```bash
eksctl get nodegroup --cluster=eksdemo1
```

#### 检查节点是否加入集群：
```bash
kubectl get nodes
```

#### 验证 IAM 角色和策略：
1. 在 AWS 管理控制台，导航到 **IAM -> Roles**。
2. 查找以 `eksdemo1-ng-public1-NodeInstanceRole` 开头的角色。
3. 检查是否附加了以下策略：
   - `AutoScalingFullAccess`
   - `AmazonEC2ContainerRegistryFullAccess`
   - `AmazonAppMeshFullAccess`
   - `ALBIngressControllerIAMPolicy`
   - `ExternalDNSIAMPolicy`

#### 验证 Kubernetes 附加组件：
1. 检查 Kubernetes 系统组件：
   ```bash
   kubectl get pods -n kube-system
   ```
2. 确认 `aws-load-balancer-controller` 和其他附加组件是否正常运行。

---

### 5. 注意事项

1. **Public Subnet**:
   - 确保集群的 VPC 有正确配置的公有子网，且子网有适当的标签：
     ```text
     kubernetes.io/role/elb = 1
     ```
   - 公有子网需要附加 Internet Gateway。

2. **安全性**:
   - 检查节点组的安全组是否允许入站流量（如 SSH 或应用程序访问）。

3. **成本控制**:
   - 动态设置节点组的最小和最大规模，避免资源浪费。

---

### 总结

通过以上命令，您可以轻松创建一个带有附加组件的托管节点组。`eksctl` 将自动为节点组生成 IAM 角色并附加所需的策略，简化了配置过程。这些附加组件（如 ALB Ingress、App Mesh 和 External DNS）提供了对 AWS 服务的原生集成，使您的 EKS 集群更加强大和灵活。如果您还有其他问题，请随时询问！

---

### 验证 Amazon EKS 集群和节点组（NodeGroup）

以下是详细步骤，帮助您验证 Amazon EKS 集群、节点组和相关配置（如公共子网、IAM 角色、安全组等）。

---

### **1. 验证集群和节点组状态**

#### 列出 EKS 集群
```bash
eksctl get cluster
```

#### 列出集群的节点组
```bash
eksctl get nodegroup --cluster=<clusterName>
```

#### 列出 Kubernetes 集群的节点
```bash
kubectl get nodes -o wide
```

#### 验证 kubectl 上下文
```bash
kubectl config view --minify
```
确保当前上下文是您创建的集群。

---

### **2. 验证节点组的子网（Public Subnet）**

1. **导航到 AWS 管理控制台**：
   - 打开 **Services -> Elastic Kubernetes Service (EKS)**。
   - 选择您的集群（`eksdemo1`）。
   - 找到节点组（`eksdemo1-ng1-public`），并点击进入。

2. **检查关联的子网**：
   - 在节点组的 **Details** 选项卡中，找到 **Associated subnet**。
   - 点击子网 ID 查看详情。

3. **验证路由表**：
   - 在子网页面中，点击 **Route Table** 选项卡。
   - 验证是否存在以下路由：
     ```
     Destination       Target
     0.0.0.0/0         igw-xxxxxxxx (Internet Gateway)
     ```
     如果看到此路由，说明子网是公共子网。

---

### **3. 验证 EKS 集群和 Worker Nodes**

#### 在 EKS 管理控制台验证
1. 打开 **Services -> Elastic Kubernetes Service**。
2. 选择您的集群（`eksdemo1`）。
3. 在 **Compute** 部分，查看节点组及其状态。
4. 查看 **Networking** 部分，确认节点组的子网是否为公共子网。

#### 列出 Worker Nodes
通过以下命令列出正在运行的 Worker Nodes：
```bash
kubectl get nodes -o wide
```

---

### **4. 验证 Worker Nodes 的 IAM 角色和安全组**

#### 验证 IAM 角色
1. 转到 **Services -> EC2 -> Instances**。
2. 找到您的 Worker Node 实例，点击实例。
3. 在 **Description** 选项卡中，点击 **IAM Role**。
4. 验证 IAM 角色是否附加了以下策略：
   - `AmazonEC2ContainerRegistryFullAccess`
   - `AmazonEKSWorkerNodePolicy`
   - `AmazonEKS_CNI_Policy`

#### 验证安全组
1. 在 **EC2 -> Instances** 中找到 Worker Node。
2. 点击实例的 **Security Groups**。
3. 确认安全组规则：
   - 入站规则：允许 SSH（端口 22）和 Kubernetes 的必要端口（如 443、10250、30000-32767）。
   - 出站规则：允许所有流量（通常为 0.0.0.0/0）。

---

### **5. 验证 CloudFormation 堆栈**

#### 验证控制平面堆栈
1. 转到 **Services -> CloudFormation**。
2. 找到与 EKS 集群相关的控制平面堆栈（通常以 `eksctl-<clusterName>` 开头）。
3. 点击堆栈名称，查看 **Events**。
   - 确认堆栈的状态为 `CREATE_COMPLETE` 或 `UPDATE_COMPLETE`。

#### 验证节点组堆栈
1. 在 CloudFormation 页面，找到节点组相关的堆栈（通常以 `eksctl-<clusterName>-nodegroup` 开头）。
2. 点击堆栈名称，查看 **Events**。
   - 确认堆栈状态为 `CREATE_COMPLETE`。

---

### **6. 登录 Worker Node 验证**

#### 登录 Worker Node
1. 获取 Worker Node 的公网 IP 地址：
   - 转到 **Services -> EC2 -> Instances**。
   - 找到您的 Worker Node，复制其 **Public IPv4 address**。

2. 使用 SSH 登录：
   ```bash
   ssh -i kube-demo.pem ec2-user@<Public-IP-of-Worker-Node>
   ```

3. 登录后，验证节点的连接性和配置：
   - 检查挂载的 EBS 卷：
     ```bash
     df -h
     ```
   - 查看运行的 Kubernetes 服务：
     ```bash
     sudo docker ps
     ```

---

### **7. 验证附加组件**

#### 验证 ALB Ingress Controller
1. 检查 ALB Ingress Controller 的状态：
   ```bash
   kubectl get pods -n kube-system | grep aws-load-balancer
   ```
   确保 Pod 正常运行。

2. 验证 ALB 是否已创建：
   - 转到 **Services -> EC2 -> Load Balancers**。
   - 确认是否有一个与您的 Ingress 资源匹配的 ALB。

#### 验证 External DNS
1. 查看 External DNS 的 Pod 状态：
   ```bash
   kubectl get pods -n kube-system | grep external-dns
   ```
   确保 Pod 正常运行。

2. 验证域名解析：
   - 在您的 DNS 服务中，检查是否已创建相应的记录。

#### 验证 Auto Scaling Group
1. 打开 **Services -> EC2 -> Auto Scaling Groups**。
2. 找到与您的节点组相关的 Auto Scaling Group。
3. 验证其最小、最大和期望实例数是否与配置匹配。

---

### **总结**

通过上述步骤，您可以验证以下内容：
1. **EKS 集群和节点组的状态**。
2. **节点组是否部署在公共子网**。
3. **IAM 角色和安全组配置**。
4. **CloudFormation 堆栈的成功创建**。
5. **Worker Node 的运行状态和连通性**。
6. **附加组件（如 ALB、External DNS）的正确运行**。

