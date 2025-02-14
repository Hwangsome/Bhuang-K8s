# EKS Storage 介绍


# Step 02: install the EBS CSI driver
## CSI Driver
### 什么是 EBS CSI Driver？

**EBS CSI Driver** 是 Amazon 提供的 **Container Storage Interface (CSI)** 插件，用于在 Kubernetes 集群（如 Amazon EKS）中动态管理 **Elastic Block Store (EBS)** 卷。它使 Kubernetes 应用程序能够使用 PVC（PersistentVolumeClaim）动态创建、挂载和管理 EBS 卷，提供了存储扩展和管理的标准化接口。

---

### 为什么需要 EBS CSI Driver？

默认情况下，Kubernetes 自带内置的 EBS 插件，但该插件存在一些局限性：

1. **功能限制**：原生的 EBS 支持有限，无法实现高级功能，如卷扩展和快照。
2. **解耦存储与核心功能**：Kubernetes 正逐步将存储相关逻辑从核心组件中解耦，CSI 是未来的标准接口。
3. **云服务深度集成**：EBS CSI Driver 提供对 AWS EBS 服务的原生支持，支持更多 AWS 存储功能。

通过安装 EBS CSI Driver，您可以：
- 动态创建和删除 EBS 卷。
- 挂载和卸载卷到 Kubernetes Pod。
- 支持卷扩展、快照和还原等高级功能。
- 与 AWS IAM 集成，实现安全的存储访问控制。

---

### EBS CSI Driver 的功能

1. **动态卷创建和管理**：
    - 自动根据 PVC 的请求创建 EBS 卷，并与 Kubernetes 的 Persistent Volume 绑定。

2. **支持高级存储功能**：
    - 卷扩展：支持在线扩展 EBS 卷的大小。
    - 快照：允许创建卷快照并恢复到新卷。
    - 卷加密：支持基于 AWS Key Management Service (KMS) 的卷加密。

3. **支持多种文件系统**：
    - 支持 `ext4` 和 `xfs` 文件系统。

4. **与 IAM 集成**：
    - 使用 Kubernetes Service Account 和 AWS IAM Roles for Service Accounts (IRSA) 控制存储访问权限。

5. **与 Kubernetes StorageClass 集成**：
    - 动态配置存储类型（如 gp2、gp3、io1）。

---

### EBS CSI Driver 的架构

1. **Controller**:
    - 运行在 Kubernetes 集群中的一个 Pod。
    - 负责处理卷的创建、扩展和删除等管理操作。
    - 通常运行在主控平面（Master Plane）上。

2. **Node Plugin**:
    - 在 Kubernetes 集群的每个节点上运行。
    - 负责处理卷的挂载和卸载。

3. **CSI Interface**:
    - 使用 CSI 标准接口与 Kubernetes 交互。

---

### 使用 EBS CSI Driver 的步骤
#### 如何在 Amazon EKS 集群中配置和使用 EBS 作为存储的详细步骤

##### EKS Storage with EBS - Elastic Block Store

以下是如何在 Amazon EKS 集群中配置和使用 EBS 作为存储的详细步骤。

---

##### **Step 1: 概述**
1. **创建 IAM Policy for EBS**：为 Amazon EBS CSI Driver 创建 IAM 策略，允许节点访问 EBS 资源。
2. **关联 IAM 策略到 Worker Node 的 IAM 角色**：将上述策略关联到节点组使用的 IAM 角色。
3. **安装 EBS CSI Driver**：在 Kubernetes 集群中安装 EBS CSI Driver，支持动态卷管理。

---

##### **Step 2: 创建 IAM 策略**

###### 1. 登录到 AWS 管理控制台：
- 打开 **Services -> IAM**。
- 点击 **Policies -> Create Policy**。

###### 2. 在 JSON 选项卡中粘贴以下策略：
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:AttachVolume",
        "ec2:CreateSnapshot",
        "ec2:CreateTags",
        "ec2:CreateVolume",
        "ec2:DeleteSnapshot",
        "ec2:DeleteTags",
        "ec2:DeleteVolume",
        "ec2:DescribeInstances",
        "ec2:DescribeSnapshots",
        "ec2:DescribeTags",
        "ec2:DescribeVolumes",
        "ec2:DetachVolume"
      ],
      "Resource": "*"
    }
  ]
}
```

###### 3. 点击 **Visual Editor** 验证策略。
- 确认策略的权限和资源。

###### 4. 点击 **Review Policy** 并填写信息：
- **Name**: `Amazon_EBS_CSI_Driver`
- **Description**: `Policy for EC2 Instances to access Elastic Block Store`

###### 5. 点击 **Create Policy**。

---

##### **Step 3: 关联 IAM 策略到 Worker Node 的 IAM 角色**

###### 1. 获取 Worker Node 使用的 IAM 角色 ARN
运行以下命令查看 `aws-auth` ConfigMap：
```bash
kubectl -n kube-system describe configmap aws-auth
```

输出示例：
```yaml
mapRoles: |
  - rolearn: arn:aws:iam::180789647333:role/eksctl-eksdemo1-nodegroup-eksdemo-NodeInstanceRole-IJN07ZKXAWNN
    username: system:node:{{EC2PrivateDNSName}}
    groups:
      - system:bootstrappers
      - system:nodes
```

记下 `rolearn` 的值（如 `arn:aws:iam::180789647333:role/eksctl-eksdemo1-nodegroup-eksdemo-NodeInstanceRole-IJN07ZKXAWNN`）。

###### 2. 在 AWS 管理控制台关联策略：
1. 打开 **Services -> IAM -> Roles**。
2. 搜索 `eksctl-eksdemo1-nodegroup` 并打开对应的角色。
3. 点击 **Permissions** 选项卡，选择 **Attach Policies**。
4. 搜索 `Amazon_EBS_CSI_Driver` 并点击 **Attach Policy**。

---

##### **Step 3-1: 可以使用aws cli 去完成step 3 的内容
以下是使用 **AWS CLI** 创建 IAM 策略的步骤和命令：

---

###### **步骤 1: 创建策略的 JSON 文件**
1. 在您的终端创建一个 JSON 文件，例如 `Amazon_EBS_CSI_Driver.json`：
   ```bash
   cat <<EOF > Amazon_EBS_CSI_Driver.json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Effect": "Allow",
         "Action": [
           "ec2:AttachVolume",
           "ec2:CreateSnapshot",
           "ec2:CreateTags",
           "ec2:CreateVolume",
           "ec2:DeleteSnapshot",
           "ec2:DeleteTags",
           "ec2:DeleteVolume",
           "ec2:DescribeInstances",
           "ec2:DescribeSnapshots",
           "ec2:DescribeTags",
           "ec2:DescribeVolumes",
           "ec2:DetachVolume"
         ],
         "Resource": "*"
       }
     ]
   }
   EOF
   ```

---

###### **步骤 2: 使用 AWS CLI 创建 IAM 策略**
运行以下命令将上述策略上传到 IAM 中：

```bash
aws iam create-policy \
  --policy-name Amazon_EBS_CSI_Driver \
  --policy-document file://Amazon_EBS_CSI_Driver.json \
  --description "Policy for EC2 Instances to access Elastic Block Store"
```

---

###### **步骤 3: 验证策略创建**
运行以下命令查看创建的策略：
```bash
aws iam list-policies --query 'Policies[?PolicyName==`Amazon_EBS_CSI_Driver`]' --output table
```

输出示例：
```plaintext
-------------------------------------------------------------------
|                       List of Policies                         |
-------------------------------------------------------------------
| Arn                                   | PolicyName             |
|---------------------------------------|------------------------|
| arn:aws:iam::123456789012:policy/Amazon_EBS_CSI_Driver | Amazon_EBS_CSI_Driver |
-------------------------------------------------------------------
```

记下策略的 ARN（例如：`arn:aws:iam::123456789012:policy/Amazon_EBS_CSI_Driver`）。

---

###### **步骤 4: 关联 IAM 策略到 Worker Node 角色**
1. 找到 Worker Node 的 IAM 角色名称：
   ```bash
   kubectl -n kube-system describe configmap aws-auth
   ```
   输出中的 `rolearn` 是节点组的 IAM 角色 ARN。

2. 使用以下命令将新创建的策略附加到 Worker Node 角色：
   ```bash
   aws iam attach-role-policy \
     --role-name <WorkerNodeRoleName> \
     --policy-arn arn:aws:iam::123456789012:policy/Amazon_EBS_CSI_Driver
   ```

   **注意**：将 `<WorkerNodeRoleName>` 替换为实际的 IAM 角色名称。

---

###### **步骤 5: 验证策略是否附加成功**
运行以下命令查看角色的策略：
```bash
aws iam list-attached-role-policies --role-name <WorkerNodeRoleName>
```

输出示例：
```plaintext
{
    "AttachedPolicies": [
        {
            "PolicyName": "AmazonEKSWorkerNodePolicy",
            "PolicyArn": "arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy"
        },
        {
            "PolicyName": "Amazon_EBS_CSI_Driver",
            "PolicyArn": "arn:aws:iam::123456789012:policy/Amazon_EBS_CSI_Driver"
        }
    ]
}
```

---


##### **Step 4: 部署 Amazon EBS CSI Driver**

###### 1. 验证 kubectl 版本
确保 Kubernetes 客户端版本在 1.14 或更高：
```bash
kubectl version --client --short
```

###### 2. 部署 EBS CSI Driver
运行以下命令部署 Amazon EBS CSI Driver：
```bash
kubectl apply -k "github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable/?ref=master"
```

###### 3. 验证 EBS CSI Driver 的 Pods
查看 EBS CSI Driver 的 Pods 状态：
```bash
kubectl get pods -n kube-system
```

输出示例：
```plaintext
NAME                                     READY   STATUS    RESTARTS   AGE
ebs-csi-controller-0                     2/2     Running   0          1m
ebs-csi-node-xyz123                      3/3     Running   0          1m
```

---

##### 验证流程完成后，EBS CSI Driver 已成功安装并运行，可以动态管理 Amazon EBS 卷。

##### 接下来：
1. 创建 **StorageClass**、**PVC** 和挂载 EBS 卷到 Pod。
2. 动态分配 EBS 存储以支持工作负载。需要帮助时，请随时告知！

### 总结

**EBS CSI Driver** 是 Kubernetes 集成 Amazon EBS 的核心插件，为 EKS 集群提供动态存储管理功能。它通过标准的 CSI 接口简化了存储管理，支持高级功能如卷扩展和快照，适用于需要高性能块存储的工作负载（如数据库和日志）。结合 Kubernetes 的 PVC 和 PV 机制，EBS CSI Driver 提供了灵活、安全和高效的存储解决方案。

# Step 03: Create Kubernetes Manifests for StorageClass and PersistentVolumeClaim and ConfigMap

# Step 04: Create Kubernetes Manifests for MySQL Deployment and Cluster Service

# Step 05: Test by connecting to the MySQL database

# Step 06: StorageClass Reference

# Step 07: Create Kubernetes Manifests for User Management Microservice Deployment 

# Step 08: Test User Management Microservice with Mysql Database in Kubernetes

# Step 09: Test User Management Microservice UMS using Postman


