# 服务发布入门

## 在K8s中是如何发布服务的

## 什么是Label 和 Selector
**Labels** 和 **Selectors** 是 Kubernetes 中的关键概念，用于组织、标识和选择 Kubernetes 对象（如 Pod、Service、Deployment 等）。它们的核心功能是帮助 Kubernetes 系统和用户高效地管理、选择和操作对象。

### **1. 什么是 Label？**

**Label** 是一种键值对（key-value pair），用于附加到 Kubernetes 资源上（如 Pod、Service、Node 等），帮助标识这些资源的属性。Label 可以用于组织、分类和查询这些资源，但 Label 不具有唯一性，多个资源可以共享相同的 Label。
```shell
kubectl get po --show-labels
NAME                                READY   STATUS    RESTARTS   AGE   LABELS
nginx-deployment-6f78d88448-9n6gn   1/1     Running   0          23h   app=nginx,pod-template-hash=6f78d88448
nginx-deployment-6f78d88448-s6zkn   1/1     Running   0          23h   app=nginx,pod-template-hash=6f78d88448
nginx-deployment-6f78d88448-xmv9s   1/1     Running   0          23h   app=nginx,pod-template-hash=6f78d88448
```

#### **Label 的特点**：
- **键值对格式**：`key: value`，如 `app: nginx` 或 `env: production`。
- **多个 Label**：每个对象可以拥有多个不同的 Label 来描述其不同的属性。
- **用户自定义**：Label 是用户定义的，Kubernetes 系统不会对其内容进行严格约束，用户可以根据需要自定义标签以适应不同的工作场景。

#### **Label 的示例**：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
    env: production
spec:
  containers:
  - name: nginx
    image: nginx
```

在这个示例中，`nginx-pod` 这个 Pod 被打上了两个 Label：`app: nginx` 和 `env: production`。

### **2. 什么是 Selector？**

**Selector** 是 Kubernetes 中用来选择带有特定 Label 的对象的机制。通过 Selector，可以查询和筛选符合特定条件的资源。Selector 分为两种类型：**等值匹配（Equality-based selector）** 和 **集合匹配（Set-based selector）**。

#### **Selector 的类型**：

1. **等值匹配（Equality-based selector）**：
    - 等值匹配是根据 Label 的键值对进行匹配。可以指定 `=`（等于）或 `!=`（不等于）条件，来选择符合这些条件的对象。
    - 例如，`app=nginx` 选择所有 Label 中包含 `app=nginx` 的 Pod。

2. **集合匹配（Set-based selector）**：
    - 集合匹配允许通过集合操作（如 `in`、`notin` 等）来选择符合某些条件的对象。它可以对 Label 的值进行更灵活的匹配。
    - 例如，`app in (nginx, apache)` 选择所有 `app` 为 `nginx` 或 `apache` 的 Pod。

#### **Selector 的示例**：
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```

在这个示例中，`nginx-service` 通过 `selector: app: nginx` 来选择所有带有 `app=nginx` 标签的 Pod，并将流量转发给这些 Pod。

### **3. Label 和 Selector 的应用场景**

- **服务发现**：Kubernetes Service 使用 Label Selector 来定位它们所关联的 Pod。Service 通过选择带有特定 Label 的 Pod，确保流量只被转发到这些 Pod。

- **滚动更新**：在 Deployment 中，通过 Label 可以区分新版本和旧版本的 Pod。Selector 用于识别需要更新的 Pod 副本。

- **批量操作**：使用 Label 可以方便地对一组资源进行批量操作。例如，删除所有 `env=production` 的 Pod。

- **监控与管理**：Label 还可以用来为不同环境、不同版本的资源分类，帮助系统管理员更好地监控和管理集群。

### **4. Label 和 Selector 的优势**

- **灵活性**：Label 可以自由定义，且可以打在任何对象上，使得它成为一种高度灵活的资源管理方式。
- **可扩展性**：通过 Label 和 Selector，用户可以轻松扩展和调整不同的服务架构，而无需修改具体的 Pod、Service 等对象配置。
- **易用性**：通过简单的 Selector 语法，用户可以轻松筛选和管理不同的资源对象。

### **总结**

- **Label** 是一种简单的键值对，用来为 Kubernetes 资源打标签，标识这些资源的属性。
- **Selector** 是一种查询机制，用来选择带有特定 Label 的资源对象，以实现动态筛选和管理。
- Label 和 Selector 的结合在 Kubernetes 中有广泛的应用场景，如服务发现、滚动更新、批量操作和集群监控等。

这些功能使 Kubernetes 具备了强大的资源管理能力，帮助用户高效管理大规模的集群.

## Label 和 Selector 的使用
**Label** 和 **Selector** 是 Kubernetes 中用于标识和筛选资源对象的重要机制。它们广泛应用于管理、查询、批量操作等场景，帮助用户灵活控制和操作 Kubernetes 资源。以下是 Label 和 Selector 的具体使用方法。

### **1. 使用 Label 给资源对象打标签**
```shell
 kubectl label -h
Update the labels on a resource.

  *  A label key and value must begin with a letter or number, and may contain letters, numbers, hyphens, dots, and
underscores, up to 63 characters each.
  *  Optionally, the key can begin with a DNS subdomain prefix and a single '/', like example.com/my-app.
  *  If --overwrite is true, then existing labels can be overwritten, otherwise attempting to overwrite a label will
result in an error.
  *  If --resource-version is specified, then updates will use this resource version, otherwise the existing
resource-version will be used.

Examples:
  # Update pod 'foo' with the label 'unhealthy' and the value 'true'
  kubectl label pods foo unhealthy=true
  
  # Update pod 'foo' with the label 'status' and the value 'unhealthy', overwriting any existing value
  kubectl label --overwrite pods foo status=unhealthy
  
  # Update all pods in the namespace
  kubectl label pods --all status=unhealthy
  
  # Update a pod identified by the type and name in "pod.json"
  kubectl label -f pod.json status=unhealthy
  
  # Update pod 'foo' only if the resource is unchanged from version 1
  kubectl label pods foo status=unhealthy --resource-version=1
  
  # Update pod 'foo' by removing a label named 'bar' if it exists
  # Does not require the --overwrite flag
  # 删除标签
  kubectl label pods foo bar-

Options:
    --all=false:
        Select all resources, in the namespace of the specified resource types

    -A, --all-namespaces=false:
        If true, check the specified action in all namespaces.

    --allow-missing-template-keys=true:
        If true, ignore any errors in templates when a field or map key is missing in the template. Only applies to
        golang and jsonpath output formats.

    --dry-run='none':
        Must be "none", "server", or "client". If client strategy, only print the object that would be sent, without
        sending it. If server strategy, submit server-side request without persisting the resource.

    --field-manager='kubectl-label':
        Name of the manager used to track field ownership.

    --field-selector='':
        Selector (field query) to filter on, supports '=', '==', and '!='.(e.g. --field-selector
        key1=value1,key2=value2). The server only supports a limited number of field queries per type.

    -f, --filename=[]:
        Filename, directory, or URL to files identifying the resource to update the labels

    -k, --kustomize='':
        Process the kustomization directory. This flag can't be used together with -f or -R.

    --list=false:
        If true, display the labels for a given resource.

    --local=false:
        If true, label will NOT contact api-server but run locally.

    -o, --output='':
        Output format. One of: (json, yaml, name, go-template, go-template-file, template, templatefile, jsonpath,
        jsonpath-as-json, jsonpath-file).

    --overwrite=false:
        If true, allow labels to be overwritten, otherwise reject label updates that overwrite existing labels.

    -R, --recursive=false:
        Process the directory used in -f, --filename recursively. Useful when you want to manage related manifests
        organized within the same directory.

    --resource-version='':
        If non-empty, the labels update will only succeed if this is the current resource-version for the object. Only
        valid when specifying a single resource.

    -l, --selector='':
        Selector (label query) to filter on, supports '=', '==', and '!='.(e.g. -l key1=value1,key2=value2). Matching
        objects must satisfy all of the specified label constraints.

    --show-managed-fields=false:
        If true, keep the managedFields when printing objects in JSON or YAML format.

    --template='':
        Template string or path to template file to use when -o=go-template, -o=go-template-file. The template format
        is golang templates [http://golang.org/pkg/text/template/#pkg-overview].

Usage:
  kubectl label [--overwrite] (-f FILENAME | TYPE NAME) KEY_1=VAL_1 ... KEY_N=VAL_N [--resource-version=version]
[options]

Use "kubectl options" for a list of global command-line options (applies to all commands).

```

**Label** 是键值对，用来描述 Kubernetes 对象的属性。你可以在创建 Pod、Service、Deployment 等资源时为它们添加 Label。

#### **为 Pod 添加 Label**：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  labels:
    app: nginx
    tier: frontend
    environment: production
spec:
  containers:
  - name: nginx
    image: nginx
```

- **`app: nginx`**：表示这个 Pod 属于 `nginx` 应用。
- **`tier: frontend`**：表示它处于前端层。
- **`environment: production`**：表示它是生产环境的 Pod。

#### **动态添加 Label 到已有对象**：
可以使用 `kubectl label` 命令向已有资源添加 Label：
```bash
kubectl label pod my-pod version=v1
```

这将向 `my-pod` 添加一个 `version=v1` 的标签。


#### **1. 删除标签**

要删除一个现有的标签，可以使用 `kubectl label` 命令，并在标签的键名后加上 `-`（减号），以表明要删除该标签。

##### **删除标签的命令格式**：
```bash
kubectl label <resource-type> <resource-name> <label-key>-
```

##### **删除标签示例**：
假设你有一个名为 `my-pod` 的 Pod，并且它有一个标签 `version=v1`，现在你想要删除该标签：

```bash
kubectl label pod my-pod version-
```

这个命令将删除 `my-pod` 的 `version` 标签。

##### **批量删除带有特定标签的资源的标签**：
你也可以通过 Selector 来批量删除特定资源的标签。例如，删除所有带有 `app=nginx` 标签的 Pod 的 `version` 标签：

```bash
kubectl label pods --selector app=nginx version-
```

这个命令会删除所有带有 `app=nginx` 标签的 Pod 的 `version` 标签。

#### **2. 修改标签**

要修改一个标签，你可以通过 `kubectl label` 命令重新为资源设置标签的值。如果标签已经存在，新的值将覆盖旧的值。

##### **修改标签的命令格式**：
```bash
kubectl label <resource-type> <resource-name> <label-key>=<new-value> --overwrite
```

##### **修改标签示例**：
假设你有一个名为 `my-pod` 的 Pod，并且它的 `version` 标签值是 `v1`，现在你想将它修改为 `v2`：

```bash
kubectl label pod my-pod version=v2 --overwrite
```

通过 `--overwrite` 选项，Kubernetes 将覆盖已存在的 `version` 标签，并将其值更新为 `v2`。

##### **批量修改标签**：
同样可以使用 Selector 批量修改标签。例如，将所有带有 `app=nginx` 标签的 Pod 的 `version` 标签修改为 `v2`：

```bash
kubectl label pods --selector app=nginx version=v2 --overwrite
```

这个命令将所有带有 `app=nginx` 标签的 Pod 的 `version` 标签更新为 `v2`。

#### **3. 验证标签的删除或修改**

可以使用 `kubectl get` 命令来查看资源的标签，确保标签删除或修改操作成功。

##### **查看特定资源的标签**：
```bash
kubectl get pod my-pod --show-labels
```

##### **列出带有特定标签的资源**：
```bash
kubectl get pods --selector app=nginx --show-labels
```


### **2. 使用 Selector 筛选资源**

**Selector** 用于根据 Label 来选择 Kubernetes 对象。你可以在多种资源（如 Service、Deployment 等）中使用 Selector。

#### **等值匹配 Selector 示例**：
在 Kubernetes Service 中，Selector 可以选择与其关联的 Pod：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: nginx
    environment: production
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

这个 Service 通过 `selector` 选择带有 `app=nginx` 和 `environment=production` 标签的所有 Pod，并将流量转发给它们。
```shell
kubectl get svc -l 'app=nginx, environment=production'
```

#### **集合匹配 Selector 示例**：
使用集合操作符选择一组 Pod。例如，选择 `app` 值为 `nginx` 或 `apache` 的所有 Pod：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: multi-app-deployment
spec:
  selector:
    matchExpressions:
    - key: app
      operator: In
      values:
      - nginx
      - apache
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx-container
        image: nginx
```

在这个示例中，Deployment 使用集合匹配 `In` 操作符选择具有 `app=nginx` 或 `app=apache` 标签的 Pod。
```shell
kubectl get po -l 'app in (nginx, apache)'
```

### **3. 批量操作资源**

使用 `kubectl` 和 Label，可以批量选择并操作资源。

#### **列出带特定 Label 的 Pod**：
```bash
kubectl get pods --selector app=nginx
```
这个命令将列出所有带有 `app=nginx` 标签的 Pod。

#### **删除带特定 Label 的资源**：
```bash
kubectl delete pods --selector app=nginx
```
该命令会删除所有带有 `app=nginx` 标签的 Pod。

### **4. 实际应用场景**

- **服务发现**：Kubernetes Service 使用 Selector 来确定应将流量转发到哪些 Pod。例如，前端服务可能只需要转发请求到标记为 `app=nginx` 的 Pod。

- **滚动更新**：在 Deployment 中，使用 Label 和 Selector 可以实现逐步替换旧版本 Pod 为新版本的 Pod，确保应用无缝更新。

- **环境隔离**：Label 可以用于标识不同环境（如 `production`、`staging`），便于操作和监控。例如，可以通过 Label 轻松隔离测试和生产环境的资源。

### **总结**

- **Label** 是一种用于标识 Kubernetes 对象的键值对，它可以为每个资源对象添加多维属性。
- **Selector** 是 Kubernetes 中选择带有特定 Label 的资源对象的机制，支持等值匹配和集合匹配。
- Label 和 Selector 结合，提供了一种灵活的方式来组织、筛选、操作和监控 Kubernetes 中的大量资源对象，极大简化了集群管理的复杂性.

## 什么是 service
在 Kubernetes 中，**Service** 是一种资源对象，主要用于定义一组 **Pod** 的访问方式，并为这些 Pod 提供持久、统一的网络服务。Pod 是动态的、短暂的，当它们重新调度时，它们的 IP 地址会发生变化。**Service** 解决了 Pod 的这种动态特性，通过提供一个稳定的 IP 地址和 DNS 名称，使得外部或集群内的应用可以一致地访问这些 Pod。

### **Kubernetes Service 的主要功能**

1. **稳定的访问入口**：即使 Pod 的 IP 地址发生变化，Service 仍然提供一个固定的 Cluster IP，使外部和集群内部的客户端可以通过这个固定的 IP 来访问 Pod。

2. **负载均衡**：Service 会自动将客户端请求分发到与该 Service 关联的多个 Pod。通过负载均衡，可以在多个 Pod 之间分配流量，从而提高应用的可用性和性能。

3. **服务发现**：通过 Kubernetes 的 DNS 系统，Service 可以自动为与之关联的 Pod 提供 DNS 名称解析。每个 Service 都有一个 DNS 名称，客户端可以通过这个名称来访问关联的 Pod。

### **Service 的类型**

Kubernetes 支持多种类型的 Service，适用于不同的网络场景：

1. **ClusterIP**（默认类型）：
    - 为 Service 分配一个集群内部的虚拟 IP（Cluster IP）。只能通过集群内部访问，适用于集群内部的服务通信。
    - **示例用途**：前端应用程序与后端服务之间的通信。

2. **NodePort**：
    - 在每个节点上开放一个端口，并将该端口映射到 Cluster IP。外部可以通过 `<NodeIP>:<NodePort>` 的形式访问服务。
    - **示例用途**：当你想让外部用户通过固定端口访问服务时使用。

3. **LoadBalancer**：
    - 通过外部负载均衡器公开服务，自动分配一个外部的 IP 地址，并将流量转发到集群中的 NodePort 服务。
    - **示例用途**：适用于云环境中的应用，自动利用云提供商的负载均衡服务（如 AWS ELB、GCP Load Balancer 等）。

4. **ExternalName**：
    - 将服务映射到一个 DNS 名称。使用 `CNAME` 记录将流量重定向到外部服务。
    - **示例用途**：适用于需要访问集群外部服务的情况，例如访问外部数据库服务。

### **Service 的配置示例**

以下是一个基于 **ClusterIP** 类型的 Service 的 YAML 配置示例：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
```

- **`selector`**：定义该 Service 关联的 Pod，Kubernetes 会将流量路由到带有 `app: my-app` 标签的 Pod。
- **`ports`**：定义服务的公开端口和 Pod 的目标端口。`port` 表示服务暴露的端口，`targetPort` 表示实际 Pod 内部监听的端口。

### **Service 的工作原理**

1. **标签选择器（Label Selector）**：
    - Service 使用标签选择器将流量路由到符合条件的 Pod。当 Service 创建时，Kubernetes 控制器会根据标签选择器选择匹配的 Pod，绑定它们的 IP，并通过 Cluster IP 实现访问。

2. **Endpoints**：
    - Service 维护一个关联的 **Endpoints** 对象，存储与 Service 相关联的 Pod 的 IP 地址。当 Pod 启动或停止时，Endpoints 自动更新，使得 Service 始终能准确定位到运行中的 Pod。

## 为什么需要Service?
**Kubernetes Service** 是集群管理中的一个关键组件，它为动态的、短暂的 Pod 提供了稳定的访问入口和负载均衡功能。以下是需要 **Service** 的几个关键原因：

### **1. 稳定的访问入口**
在 Kubernetes 中，Pod 的生命周期是短暂的、动态的。当某个 Pod 被重新调度或崩溃时，它的 IP 地址会发生变化。这对外部或集群内部的应用程序来说是一个问题，因为它们无法稳定地通过 IP 地址访问 Pod。**Service** 通过为 Pod 提供一个稳定的 Cluster IP，使得即使 Pod 的 IP 发生变化，客户端也可以通过固定的 Service IP 访问应用程序。

### **2. 负载均衡**
在 Kubernetes 中，通常会运行多个相同的 Pod 副本来处理高并发请求。**Service** 提供了内置的负载均衡功能，它会将流量自动分配给与之关联的多个 Pod，确保流量分布均匀，避免单一 Pod 过载。Kubernetes Service 通过 Label 和 Selector 选择匹配的 Pod，并将流量路由到这些 Pod 上。

### **3. 支持自动服务发现**
Kubernetes 集群中存在许多微服务组件，彼此之间需要通信。**Service** 通过 Kubernetes 内置的 DNS 系统提供了服务发现功能。每个 Service 都会有一个固定的 DNS 名称，其他服务或 Pod 可以通过这个 DNS 名称直接访问它，而不需要知道它的 IP 地址。

### **4. 跨集群外部访问**
有些应用需要从外部访问 Kubernetes 内部的服务。**Service** 提供了不同的类型（如 NodePort 和 LoadBalancer）以支持从集群外部访问服务。例如，**NodePort** 将在每个节点上开放一个特定的端口，使得外部用户可以通过 `<NodeIP>:<NodePort>` 访问服务；而 **LoadBalancer** 则在云环境中使用外部负载均衡器，直接暴露一个外部 IP 地址供用户访问。

### **5. 抽象底层基础设施**
Kubernetes Service 提供了一种对应用透明的方式来抽象底层的基础设施细节。无论 Pod 是在哪个节点上运行，应用只需要通过 Service 就可以访问到它们，而不需要关心底层的网络拓扑结构。

### **总结**
Kubernetes 的 Service 解决了 Pod 的动态性、负载均衡、服务发现和外部访问等问题。它为集群内外的客户端提供了稳定的访问入口，极大简化了分布式应用在 Kubernetes 环境中的部署和管理.

## Service 的使用
```shell
kubectl explain svc.spec
KIND:       Service
VERSION:    v1

FIELD: spec <ServiceSpec>


DESCRIPTION:
    Spec defines the behavior of a service.
    https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#spec-and-status
    ServiceSpec describes the attributes that a user creates on a service.
    
FIELDS:
  allocateLoadBalancerNodePorts <boolean>
    allocateLoadBalancerNodePorts defines if NodePorts will be automatically
    allocated for services with type LoadBalancer.  Default is "true". It may be
    set to "false" if the cluster load-balancer does not rely on NodePorts.  If
    the caller requests specific NodePorts (by specifying a value), those
    requests will be respected, regardless of this field. This field may only be
    set for services with type LoadBalancer and will be cleared if the type is
    changed to any other type.

  clusterIP     <string>
    clusterIP is the IP address of the service and is usually assigned randomly.
    If an address is specified manually, is in-range (as per system
    configuration), and is not in use, it will be allocated to the service;
    otherwise creation of the service will fail. This field may not be changed
    through updates unless the type field is also being changed to ExternalName
    (which requires this field to be blank) or the type field is being changed
    from ExternalName (in which case this field may optionally be specified, as
    describe above).  Valid values are "None", empty string (""), or a valid IP
    address. Setting this to "None" makes a "headless service" (no virtual IP),
    which is useful when direct endpoint connections are preferred and proxying
    is not required.  Only applies to types ClusterIP, NodePort, and
    LoadBalancer. If this field is specified when creating a Service of type
    ExternalName, creation will fail. This field will be wiped when updating a
    Service to type ExternalName. More info:
    https://kubernetes.io/docs/concepts/services-networking/service/#virtual-ips-and-service-proxies

  clusterIPs    <[]string>
    ClusterIPs is a list of IP addresses assigned to this service, and are
    usually assigned randomly.  If an address is specified manually, is in-range
    (as per system configuration), and is not in use, it will be allocated to
    the service; otherwise creation of the service will fail. This field may not
    be changed through updates unless the type field is also being changed to
    ExternalName (which requires this field to be empty) or the type field is
    being changed from ExternalName (in which case this field may optionally be
    specified, as describe above).  Valid values are "None", empty string (""),
    or a valid IP address.  Setting this to "None" makes a "headless service"
    (no virtual IP), which is useful when direct endpoint connections are
    preferred and proxying is not required.  Only applies to types ClusterIP,
    NodePort, and LoadBalancer. If this field is specified when creating a
    Service of type ExternalName, creation will fail. This field will be wiped
    when updating a Service to type ExternalName.  If this field is not
    specified, it will be initialized from the clusterIP field.  If this field
    is specified, clients must ensure that clusterIPs[0] and clusterIP have the
    same value.
    
    This field may hold a maximum of two entries (dual-stack IPs, in either
    order). These IPs must correspond to the values of the ipFamilies field.
    Both clusterIPs and ipFamilies are governed by the ipFamilyPolicy field.
    More info:
    https://kubernetes.io/docs/concepts/services-networking/service/#virtual-ips-and-service-proxies

  externalIPs   <[]string>
    externalIPs is a list of IP addresses for which nodes in the cluster will
    also accept traffic for this service.  These IPs are not managed by
    Kubernetes.  The user is responsible for ensuring that traffic arrives at a
    node with this IP.  A common example is external load-balancers that are not
    part of the Kubernetes system.

  externalName  <string>
    externalName is the external reference that discovery mechanisms will return
    as an alias for this service (e.g. a DNS CNAME record). No proxying will be
    involved.  Must be a lowercase RFC-1123 hostname
    (https://tools.ietf.org/html/rfc1123) and requires `type` to be
    "ExternalName".

  externalTrafficPolicy <string>
  enum: Cluster, Cluster, Local, Local
    externalTrafficPolicy describes how nodes distribute service traffic they
    receive on one of the Service's "externally-facing" addresses (NodePorts,
    ExternalIPs, and LoadBalancer IPs). If set to "Local", the proxy will
    configure the service in a way that assumes that external load balancers
    will take care of balancing the service traffic between nodes, and so each
    node will deliver traffic only to the node-local endpoints of the service,
    without masquerading the client source IP. (Traffic mistakenly sent to a
    node with no endpoints will be dropped.) The default value, "Cluster", uses
    the standard behavior of routing to all endpoints evenly (possibly modified
    by topology and other features). Note that traffic sent to an External IP or
    LoadBalancer IP from within the cluster will always get "Cluster" semantics,
    but clients sending to a NodePort from within the cluster may need to take
    traffic policy into account when picking a node.
    
    Possible enum values:
     - `"Cluster"`
     - `"Cluster"` routes traffic to all endpoints.
     - `"Local"`
     - `"Local"` preserves the source IP of the traffic by routing only to
    endpoints on the same node as the traffic was received on (dropping the
    traffic if there are no local endpoints).

  healthCheckNodePort   <integer>
    healthCheckNodePort specifies the healthcheck nodePort for the service. This
    only applies when type is set to LoadBalancer and externalTrafficPolicy is
    set to Local. If a value is specified, is in-range, and is not in use, it
    will be used.  If not specified, a value will be automatically allocated.
    External systems (e.g. load-balancers) can use this port to determine if a
    given node holds endpoints for this service or not.  If this field is
    specified when creating a Service which does not need it, creation will
    fail. This field will be wiped when updating a Service to no longer need it
    (e.g. changing type). This field cannot be updated once set.

  internalTrafficPolicy <string>
  enum: Cluster, Local
    InternalTrafficPolicy describes how nodes distribute service traffic they
    receive on the ClusterIP. If set to "Local", the proxy will assume that pods
    only want to talk to endpoints of the service on the same node as the pod,
    dropping the traffic if there are no local endpoints. The default value,
    "Cluster", uses the standard behavior of routing to all endpoints evenly
    (possibly modified by topology and other features).
    
    Possible enum values:
     - `"Cluster"` routes traffic to all endpoints.
     - `"Local"` routes traffic only to endpoints on the same node as the client
    pod (dropping the traffic if there are no local endpoints).

  ipFamilies    <[]string>
    IPFamilies is a list of IP families (e.g. IPv4, IPv6) assigned to this
    service. This field is usually assigned automatically based on cluster
    configuration and the ipFamilyPolicy field. If this field is specified
    manually, the requested family is available in the cluster, and
    ipFamilyPolicy allows it, it will be used; otherwise creation of the service
    will fail. This field is conditionally mutable: it allows for adding or
    removing a secondary IP family, but it does not allow changing the primary
    IP family of the Service. Valid values are "IPv4" and "IPv6".  This field
    only applies to Services of types ClusterIP, NodePort, and LoadBalancer, and
    does apply to "headless" services. This field will be wiped when updating a
    Service to type ExternalName.
    
    This field may hold a maximum of two entries (dual-stack families, in either
    order).  These families must correspond to the values of the clusterIPs
    field, if specified. Both clusterIPs and ipFamilies are governed by the
    ipFamilyPolicy field.

  ipFamilyPolicy        <string>
  enum: PreferDualStack, RequireDualStack, SingleStack
    IPFamilyPolicy represents the dual-stack-ness requested or required by this
    Service. If there is no value provided, then this field will be set to
    SingleStack. Services can be "SingleStack" (a single IP family),
    "PreferDualStack" (two IP families on dual-stack configured clusters or a
    single IP family on single-stack clusters), or "RequireDualStack" (two IP
    families on dual-stack configured clusters, otherwise fail). The ipFamilies
    and clusterIPs fields depend on the value of this field. This field will be
    wiped when updating a service to type ExternalName.
    
    Possible enum values:
     - `"PreferDualStack"` indicates that this service prefers dual-stack when
    the cluster is configured for dual-stack. If the cluster is not configured
    for dual-stack the service will be assigned a single IPFamily. If the
    IPFamily is not set in service.spec.ipFamilies then the service will be
    assigned the default IPFamily configured on the cluster
     - `"RequireDualStack"` indicates that this service requires dual-stack.
    Using IPFamilyPolicyRequireDualStack on a single stack cluster will result
    in validation errors. The IPFamilies (and their order) assigned to this
    service is based on service.spec.ipFamilies. If service.spec.ipFamilies was
    not provided then it will be assigned according to how they are configured
    on the cluster. If service.spec.ipFamilies has only one entry then the
    alternative IPFamily will be added by apiserver
     - `"SingleStack"` indicates that this service is required to have a single
    IPFamily. The IPFamily assigned is based on the default IPFamily used by the
    cluster or as identified by service.spec.ipFamilies field

  loadBalancerClass     <string>
    loadBalancerClass is the class of the load balancer implementation this
    Service belongs to. If specified, the value of this field must be a
    label-style identifier, with an optional prefix, e.g. "internal-vip" or
    "example.com/internal-vip". Unprefixed names are reserved for end-users.
    This field can only be set when the Service type is 'LoadBalancer'. If not
    set, the default load balancer implementation is used, today this is
    typically done through the cloud provider integration, but should apply for
    any default implementation. If set, it is assumed that a load balancer
    implementation is watching for Services with a matching class. Any default
    load balancer implementation (e.g. cloud providers) should ignore Services
    that set this field. This field can only be set when creating or updating a
    Service to type 'LoadBalancer'. Once set, it can not be changed. This field
    will be wiped when a service is updated to a non 'LoadBalancer' type.

  loadBalancerIP        <string>
    Only applies to Service Type: LoadBalancer. This feature depends on whether
    the underlying cloud-provider supports specifying the loadBalancerIP when a
    load balancer is created. This field will be ignored if the cloud-provider
    does not support the feature. Deprecated: This field was under-specified and
    its meaning varies across implementations, and it cannot support dual-stack.
    As of Kubernetes v1.24, users are encouraged to use implementation-specific
    annotations when available. This field may be removed in a future API
    version.

  loadBalancerSourceRanges      <[]string>
    If specified and supported by the platform, this will restrict traffic
    through the cloud-provider load-balancer will be restricted to the specified
    client IPs. This field will be ignored if the cloud-provider does not
    support the feature." More info:
    https://kubernetes.io/docs/tasks/access-application-cluster/create-external-load-balancer/

  ports <[]ServicePort>
    The list of ports that are exposed by this service. More info:
    https://kubernetes.io/docs/concepts/services-networking/service/#virtual-ips-and-service-proxies

  publishNotReadyAddresses      <boolean>
    publishNotReadyAddresses indicates that any agent which deals with endpoints
    for this Service should disregard any indications of ready/not-ready. The
    primary use case for setting this field is for a StatefulSet's Headless
    Service to propagate SRV DNS records for its Pods for the purpose of peer
    discovery. The Kubernetes controllers that generate Endpoints and
    EndpointSlice resources for Services interpret this to mean that all
    endpoints are considered "ready" even if the Pods themselves are not. Agents
    which consume only Kubernetes generated endpoints through the Endpoints or
    EndpointSlice resources can safely assume this behavior.

  selector      <map[string]string>
    Route service traffic to pods with label keys and values matching this
    selector. If empty or not present, the service is assumed to have an
    external process managing its endpoints, which Kubernetes will not modify.
    Only applies to types ClusterIP, NodePort, and LoadBalancer. Ignored if type
    is ExternalName. More info:
    https://kubernetes.io/docs/concepts/services-networking/service/

  sessionAffinity       <string>
  enum: ClientIP, None
    Supports "ClientIP" and "None". Used to maintain session affinity. Enable
    client IP based session affinity. Must be ClientIP or None. Defaults to
    None. More info:
    https://kubernetes.io/docs/concepts/services-networking/service/#virtual-ips-and-service-proxies
    
    Possible enum values:
     - `"ClientIP"` is the Client IP based.
     - `"None"` - no session affinity.

  sessionAffinityConfig <SessionAffinityConfig>
    sessionAffinityConfig contains the configurations of session affinity.

  type  <string>
  enum: ClusterIP, ExternalName, LoadBalancer, NodePort
    type determines how the Service is exposed. Defaults to ClusterIP. Valid
    options are ExternalName, ClusterIP, NodePort, and LoadBalancer. "ClusterIP"
    allocates a cluster-internal IP address for load-balancing to endpoints.
    Endpoints are determined by the selector or if that is not specified, by
    manual construction of an Endpoints object or EndpointSlice objects. If
    clusterIP is "None", no virtual IP is allocated and the endpoints are
    published as a set of endpoints rather than a virtual IP. "NodePort" builds
    on ClusterIP and allocates a port on every node which routes to the same
    endpoints as the clusterIP. "LoadBalancer" builds on NodePort and creates an
    external load-balancer (if supported in the current cloud) which routes to
    the same endpoints as the clusterIP. "ExternalName" aliases this service to
    the specified externalName. Several other fields do not apply to
    ExternalName services. More info:
    https://kubernetes.io/docs/concepts/services-networking/service/#publishing-services-service-types
    
    Possible enum values:
     - `"ClusterIP"` means a service will only be accessible inside the cluster,
    via the cluster IP.
     - `"ExternalName"` means a service consists of only a reference to an
    external name that kubedns or equivalent will return as a CNAME record, with
    no exposing or proxying of any pods involved.
     - `"LoadBalancer"` means a service will be exposed via an external load
    balancer (if the cloud provider supports it), in addition to 'NodePort'
    type.
     - `"NodePort"` means a service will be exposed on one port of every node,
    in addition to 'ClusterIP' type.
```

### 格式
```shell
   kubectl create -h                              
Create a resource from a file or from stdin.

 JSON and YAML formats are accepted.

Examples:
  # Create a pod using the data in pod.json
  kubectl create -f ./pod.json
  
  # Create a pod based on the JSON passed into stdin
  cat pod.json | kubectl create -f -
  
  # Edit the data in registry.yaml in JSON then create the resource using the edited data
  kubectl create -f registry.yaml --edit -o json

Available Commands:
  clusterrole           Create a cluster role
  clusterrolebinding    Create a cluster role binding for a particular cluster role
  configmap             Create a config map from a local file, directory or literal value
  cronjob               Create a cron job with the specified name
  deployment            Create a deployment with the specified name
  ingress               Create an ingress with the specified name
  job                   Create a job with the specified name
  namespace             Create a namespace with the specified name
  poddisruptionbudget   Create a pod disruption budget with the specified name
  priorityclass         Create a priority class with the specified name
  quota                 Create a quota with the specified name
  role                  Create a role with single rule
  rolebinding           Create a role binding for a particular role or cluster role
  secret                Create a secret using a specified subcommand
  service               Create a service using a specified subcommand
  serviceaccount        Create a service account with the specified name
  token                 Request a service account token

Options:
    --allow-missing-template-keys=true:
        If true, ignore any errors in templates when a field or map key is missing in the template. Only applies to
        golang and jsonpath output formats.

    --dry-run='none':
        Must be "none", "server", or "client". If client strategy, only print the object that would be sent, without
        sending it. If server strategy, submit server-side request without persisting the resource.

    --edit=false:
        Edit the API resource before creating

    --field-manager='kubectl-create':
        Name of the manager used to track field ownership.

    -f, --filename=[]:
        Filename, directory, or URL to files to use to create the resource

    -k, --kustomize='':
        Process the kustomization directory. This flag can't be used together with -f or -R.

    -o, --output='':
        Output format. One of: (json, yaml, name, go-template, go-template-file, template, templatefile, jsonpath,
        jsonpath-as-json, jsonpath-file).

    --raw='':
        Raw URI to POST to the server.  Uses the transport specified by the kubeconfig file.

    -R, --recursive=false:
        Process the directory used in -f, --filename recursively. Useful when you want to manage related manifests
        organized within the same directory.

    --save-config=false:
        If true, the configuration of current object will be saved in its annotation. Otherwise, the annotation will
        be unchanged. This flag is useful when you want to perform kubectl apply on this object in the future.

    -l, --selector='':
        Selector (label query) to filter on, supports '=', '==', and '!='.(e.g. -l key1=value1,key2=value2). Matching
        objects must satisfy all of the specified label constraints.

    --show-managed-fields=false:
        If true, keep the managedFields when printing objects in JSON or YAML format.

    --template='':
        Template string or path to template file to use when -o=go-template, -o=go-template-file. The template format
        is golang templates [http://golang.org/pkg/text/template/#pkg-overview].

    --validate='strict':
        Must be one of: strict (or true), warn, ignore (or false).              "true" or "strict" will use a schema to validate
        the input and fail the request if invalid. It will perform server side validation if ServerSideFieldValidation
        is enabled on the api-server, but will fall back to less reliable client-side validation if not.                "warn" will
        warn about unknown or duplicate fields without blocking the request if server-side field validation is enabled
        on the API server, and behave as "ignore" otherwise.            "false" or "ignore" will not perform any schema
        validation, silently dropping any unknown or duplicate fields.

    --windows-line-endings=false:
        Only relevant if --edit=true. Defaults to the line ending native to your platform.

Usage:
  kubectl create -f FILENAME [options]

Use "kubectl create <command> --help" for more information about a given command.
Use "kubectl options" for a list of global command-line options (applies to all commands).

# 获取这个命令的帮助
kubectl create service -h                      
Create a service using a specified subcommand.

Aliases:
service, svc

Available Commands:
  clusterip      Create a ClusterIP service
  externalname   Create an ExternalName service
  loadbalancer   Create a LoadBalancer service
  nodeport       Create a NodePort service

Usage:
  kubectl create service [flags] [options]

Use "kubectl create service <command> --help" for more information about a given command.
Use "kubectl options" for a list of global command-line options (applies to all commands).
bhuang@EXPJ526RGHJ02 Bhuang-k8s % kubectl create service nodeport --dry-run=client -o yaml
error: exactly one NAME is required, got 0
See 'kubectl create service nodeport -h' for help and examples
bhuang@EXPJ526RGHJ02 Bhuang-k8s % kubectl create service nodeport -h                      
Create a NodePort service with the specified name.

Examples:
  # Create a new NodePort service named my-ns
  kubectl create service nodeport my-ns --tcp=5678:8080

Options:
    --allow-missing-template-keys=true:
        If true, ignore any errors in templates when a field or map key is missing in the template. Only applies to
        golang and jsonpath output formats.

    --dry-run='none':
        Must be "none", "server", or "client". If client strategy, only print the object that would be sent, without
        sending it. If server strategy, submit server-side request without persisting the resource.

    --field-manager='kubectl-create':
        Name of the manager used to track field ownership.

    --node-port=0:
        Port used to expose the service on each node in a cluster.

    -o, --output='':
        Output format. One of: (json, yaml, name, go-template, go-template-file, template, templatefile, jsonpath,
        jsonpath-as-json, jsonpath-file).

    --save-config=false:
        If true, the configuration of current object will be saved in its annotation. Otherwise, the annotation will
        be unchanged. This flag is useful when you want to perform kubectl apply on this object in the future.

    --show-managed-fields=false:
        If true, keep the managedFields when printing objects in JSON or YAML format.

    --tcp=[]:
        Port pairs can be specified as '<port>:<targetPort>'.

    --template='':
        Template string or path to template file to use when -o=go-template, -o=go-template-file. The template format
        is golang templates [http://golang.org/pkg/text/template/#pkg-overview].

    --validate='strict':
        Must be one of: strict (or true), warn, ignore (or false).              "true" or "strict" will use a schema to validate
        the input and fail the request if invalid. It will perform server side validation if ServerSideFieldValidation
        is enabled on the api-server, but will fall back to less reliable client-side validation if not.                "warn" will
        warn about unknown or duplicate fields without blocking the request if server-side field validation is enabled
        on the API server, and behave as "ignore" otherwise.            "false" or "ignore" will not perform any schema
        validation, silently dropping any unknown or duplicate fields.

Usage:
  kubectl create service nodeport NAME [--tcp=port:targetPort] [--dry-run=server|client|none] [options]

Use "kubectl options" for a list of global command-line options (applies to all commands).


# 获取格式
 kubectl create service nodeport my-ns --tcp=5678:8080 --dry-run=client -o yaml
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  labels:
    app: my-ns
  name: my-ns
spec:
  ports:
  - name: 5678-8080
    port: 5678
    protocol: TCP
    targetPort: 8080
  selector:
    app: my-ns
  type: NodePort
status:
  loadBalancer: {}

```

## Service 的类型
在 Kubernetes 中，**Service** 是用于暴露应用的一种资源对象，提供稳定的网络访问，并帮助管理 Pod 的动态生命周期。根据不同的应用场景和需求，Kubernetes 提供了多种类型的 Service，每种类型都有其适用的使用场景。以下是 Kubernetes Service 的几种主要类型及其应用场景：

### **1. ClusterIP (默认类型)**

**ClusterIP** 是 Kubernetes Service 的默认类型，它为服务分配一个集群内部的 IP（即 Cluster IP）。此 IP 只能在 Kubernetes 集群内部访问，因此 ClusterIP 服务不能被外部流量直接访问。

#### **应用场景**：
- **集群内部通信**：当多个服务在同一个 Kubernetes 集群中运行时，集群内的服务可以通过 ClusterIP 类型的 Service 进行内部通信。例如，前端服务通过 ClusterIP 与后端服务进行通信。
- **微服务架构**：在微服务架构中，各个微服务通过 ClusterIP 相互访问，所有的服务发现和通信都通过 Kubernetes 内部的 DNS 系统自动进行。

#### **示例**：
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
  type: ClusterIP
```

### **2. NodePort**

**NodePort** 是一种可以将服务暴露到集群外部的 Service 类型。NodePort 服务会在每个 Kubernetes 节点上开放一个特定的端口（30000-32767 范围内），从而允许外部用户通过 `<NodeIP>:<NodePort>` 形式访问服务。

#### **应用场景**：
- **简单外部访问**：当你不使用云负载均衡或其他外部服务时，NodePort 提供了一种基础的方式来暴露服务，使得外部用户能够访问集群中的服务。
- **开发和测试环境**：适合在开发或测试环境中用于调试和验证外部访问的配置。

#### **示例**：
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: NodePort
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
      nodePort: 30007
```

### **3. LoadBalancer**

**LoadBalancer** Service 类型通常用于云环境中，它为服务创建一个外部的负载均衡器（例如 AWS ELB、GCP Load Balancer）。在云平台中，LoadBalancer 会为服务分配一个外部 IP，所有到达该 IP 的流量将被分发到 Kubernetes 集群内的后端 Pod。

#### **应用场景**：
- **云环境下的公开服务**：在云环境中，当你需要将服务公开给外部用户时，LoadBalancer 是最常用的方式，适用于生产环境。
- **自动负载均衡**：对于高并发的应用，LoadBalancer 会根据云提供商的负载均衡策略自动将流量分配到 Kubernetes 集群内的多个 Pod。

#### **示例**：
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-loadbalancer-service
spec:
  type: LoadBalancer
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
```

### **4. ExternalName**

**ExternalName** 类型的 Service 不会创建 Cluster IP 或 NodePort，它通过将服务的名称映射到一个外部的 DNS 名称上，从而将流量重定向到集群外部的服务。此类型的服务不进行负载均衡，也不处理流量，只是简单的 DNS 名称映射。

#### **应用场景**：
- **访问外部服务**：当 Kubernetes 集群中的应用程序需要访问集群外部的资源时（如外部数据库、第三方服务），ExternalName 可以作为一个代理，将流量重定向到外部服务的 DNS 名称。
- **统一服务访问方式**：在集群内外服务的访问方式一致，客户端不需要知道服务是在集群内还是外部。

#### **示例**：
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-externalname-service
spec:
  type: ExternalName
  externalName: external.example.com
```

### **5. Headless Service**

**Headless Service** 是一种特殊的 ClusterIP 服务，当 `clusterIP` 设置为 `None` 时，Kubernetes 不会为该服务分配一个 Cluster IP。Headless Service 通过直接暴露与之关联的 Pod 的 IP 地址，使得客户端可以通过 DNS 直接访问每个 Pod。它常用于有状态应用，如 StatefulSet，确保每个 Pod 都有一个稳定的网络标识。

#### **应用场景**：
- **有状态服务（StatefulSet）**：在有状态服务中（如数据库集群、分布式存储），Headless Service 可确保每个 Pod 都有独立的网络标识。
- **Pod 间直接通信**：允许客户端通过 DNS 直接解析并访问特定的 Pod，适用于需要明确与某个 Pod 通信的场景。

#### **示例**：
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
      targetPort: 9376
```

### **总结**

Kubernetes 中的 **Service** 类型丰富，适应不同的应用场景：
- **ClusterIP**：用于集群内部的服务通信。
- **NodePort**：用于集群外部用户通过节点访问服务，适合简单的外部访问场景。
- **LoadBalancer**：为服务创建外部负载均衡器，适合云环境中的生产级应用。
- **ExternalName**：用于将集群内的服务映射到外部服务的 DNS 名称。
- **Headless Service**：用于有状态应用的直接 Pod 访问场景。

选择合适的 Service 类型，能够为应用提供最佳的可扩展性和灵活性【30†source】【32†source】.

## 使用NodePort 对外发布服务
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 90         # service 暴露的端口
      targetPort: 80   # 容器内的端口
      nodePort: 30000  # 可指定，也可让系统自动分配 The range of valid ports is 30000-32767
  type: NodePort       # 使用 NodePort 类型
```

## NodePort 的工作原理
**NodePort** 是 Kubernetes 中一种 Service 类型，它的作用是通过在集群每个节点上开放一个固定端口，允许集群外部的用户访问 Kubernetes 集群内的服务。NodePort 服务将外部的请求通过集群中的每个节点转发到相应的 Pod。

### **NodePort 的工作原理**

1. **NodePort 的定义**：
    - 当你在 Kubernetes 中创建一个 `NodePort` 类型的 Service 时，Kubernetes 会在集群中每个节点的特定范围内的端口（30000-32767）开放一个端口。这个端口称为 **NodePort**。
    - 所有节点上的这个端口都映射到 Service 中定义的 Pod 的目标端口（`targetPort`）。

2. **流量转发**：
    - 当外部用户访问 `<NodeIP>:<NodePort>` 时，Kubernetes 会将请求路由到相应的 Pod 上，无论 Pod 实际运行在哪个节点上。
    - 这意味着即使 Pod 运行在不同的节点上，用户仍可以通过任意一个节点的 IP 和 NodePort 端口来访问 Pod。

3. **内置负载均衡**：
    - NodePort 本质上是 Kubernetes 服务的一部分，它会通过 Kubernetes 内置的负载均衡器（基于 iptables 或 IPVS）在所有关联的 Pod 之间分配请求。
    - 通过任何节点访问 NodePort，Kubernetes 会将流量路由到运行该应用程序的 Pod。

4. **拓展 NodePort 的外部访问**：
    - NodePort 通常与 **LoadBalancer** 类型的 Service 结合使用，特别是在云平台上。当 LoadBalancer 创建时，它实际上使用 NodePort 作为内部机制，并在云环境中将负载均衡流量转发到 NodePort 服务。

### **NodePort 的配置和使用**

#### **NodePort Service 配置示例**：
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-nodeport-service
spec:
  type: NodePort
  selector:
    app: my-app
  ports:
    - port: 80        # Service 暴露的端口
      targetPort: 8080 # Pod 中的实际端口
      nodePort: 30007  # 在节点上开放的端口
```

- **`port: 80`**：表示客户端通过 Service 访问的端口。
- **`targetPort: 8080`**：表示 Pod 内部应用监听的端口。
- **`nodePort: 30007`**：表示集群中每个节点上的端口号，通过这个端口，外部流量可以访问 Service。

#### **访问方式**：
外部用户可以通过 `<NodeIP>:30007` 的形式访问该服务，无论流量进入哪个节点，Kubernetes 会自动路由流量到实际运行 Pod 的节点上。

### **NodePort 的应用场景**

1. **小规模集群或开发环境**：NodePort 通常用于小型集群或测试环境，特别是在无需复杂的云负载均衡器的情况下提供简单的外部访问。

2. **简单的外部访问**：NodePort 提供了一种快速的方式来开放服务给外部用户，允许他们通过节点的 IP 地址直接访问集群内部的服务。

3. **负载均衡器的内部机制**：NodePort 是 Kubernetes 中 **LoadBalancer** 服务的基础，当你在云环境中使用 LoadBalancer 服务时，实际上也是在使用 NodePort 来实现外部流量进入集群内部。

### **NodePort 的限制**

- **端口范围限制**：NodePort 服务只能使用 30000 到 32767 之间的端口，因此可用端口有限。
- **安全性**：由于 NodePort 在每个节点上开放端口，可能增加攻击面，不适合直接用于生产环境的大规模服务。
- **手动配置外部负载均衡**：与 LoadBalancer 类型不同，NodePort 不会自动配置外部负载均衡器，你可能需要在集群外部手动配置负载均衡器。

### **总结**

- **NodePort** 允许 Kubernetes 集群中的服务通过每个节点的固定端口被外部访问。它通过在集群每个节点上开放一个端口，将流量转发到 Service 关联的 Pod 中，并通过内置的负载均衡器在多个 Pod 之间分发流量。
- NodePort 适合小规模集群、开发和测试环境，也可以用于搭建自定义的外部负载均衡解决方案。

## nodeport 是怎么映射到pod 的 targetPort
在 Kubernetes 中，**NodePort** 服务通过固定端口将外部流量引入集群，并将这些流量转发到集群内部的 **Pod** 上的特定端口（**targetPort**）。这个过程是通过 Kubernetes 的 **kube-proxy** 组件来实现的，它管理集群中的服务与 Pod 之间的网络路由规则。

### **NodePort 到 Pod targetPort 的映射过程**

1. **外部流量进入节点的 NodePort**：
    - 当外部用户通过 `NodeIP:NodePort` 的方式访问服务时，请求首先到达 Kubernetes 集群中的某个节点的 NodePort 端口。NodePort 是一个从 30000 到 32767 的固定范围内的端口。
    - 这个 NodePort 是对外暴露的入口，无论客户端请求哪个节点的 NodePort，Kubernetes 都会将其转发到正确的 Pod。

2. **kube-proxy 路由流量**：
    - 每个 Kubernetes 节点上运行的 **kube-proxy** 负责管理服务的网络规则。kube-proxy 通过 **iptables** 或 **IPVS** 实现流量的转发。它会监听 NodePort，并根据配置好的规则，将进入 NodePort 的流量转发给与服务关联的 Pod。
    - kube-proxy 通过服务的 **Selector** 将流量定向到具有特定 Label 的 Pod。它会将流量分配到这些符合条件的 Pod 上，并在多个 Pod 之间进行负载均衡。

3. **Service 的端口映射（NodePort -> port -> targetPort）**：
    - **NodePort** 服务定义了三类端口：
        - **`nodePort`**：用于外部流量进入节点的端口（30000-32767）。
        - **`port`**：Service 的集群内部端口（通常是 ClusterIP 类型服务使用的端口），但也可以通过 NodePort 来映射。该端口可以与 targetPort 不同。
        - **`targetPort`**：Pod 内部应用实际监听的端口。kube-proxy 将流量从 `port` 转发到 Pod 中监听的 `targetPort`。

    - 流量进入 NodePort 后，kube-proxy 会将流量转发到 Service 的 `port`，然后再将其路由到 Pod 中的 `targetPort`，即 Pod 实际上监听的端口。

    - **映射过程**：
        - **外部请求**：客户端通过 `NodeIP:NodePort` 访问服务。
        - **kube-proxy 转发**：kube-proxy 接收到 NodePort 的流量，将其转发到与服务关联的 Pod 中。
        - **目标 Pod 和 targetPort**：流量最终进入 Pod，并到达 Pod 内部监听的 `targetPort`。

### **具体示例**

假设定义了如下 NodePort 服务：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-nodeport-service
spec:
  type: NodePort
  selector:
    app: my-app
  ports:
    - port: 80          # Service 暴露的端口 (Cluster IP 使用)
      targetPort: 8080   # Pod 内部应用的实际监听端口
      nodePort: 30007    # 外部访问的节点端口
```

在这个配置中：
- 外部客户端通过 `NodeIP:30007` 访问服务。
- kube-proxy 在每个节点上监听端口 30007，将进入的流量路由到 `port: 80`。
- 流量接着被路由到目标 Pod，并传递给 Pod 内部监听的 `targetPort: 8080`。

通过这种端口映射机制，NodePort 服务将外部请求最终引导到正确的 Pod 上的实际端口。

### **总结**

- **NodePort** 服务提供外部访问 Kubernetes 内部 Pod 的入口，kube-proxy 负责管理流量的路由规则。
- 流量从外部进入 NodePort 端口后，经过 `port`（可选）转发到 Pod 内部应用监听的 `targetPort`。
- kube-proxy 使用 iptables 或 IPVS 规则将流量转发到符合 Selector 条件的 Pod 上，并实现负载均衡。

这种机制确保了外部流量能够通过节点的固定端口稳定地访问 Kubernetes 集群内的应用.

## 使用 Service 代理 k8s 外部服务
### k8s内的服务访问aws 的rds
在 Kubernetes 中，你可以结合 **Service** 和 **Endpoints** 来代理外部服务，如 AWS 的 RDS 数据库。由于 AWS RDS 是集群外部的服务，通过直接创建 Service 和相应的 Endpoints，可以将 Kubernetes 内部的请求路由到 AWS RDS 的外部地址。

### **配置步骤**

1. **创建 Endpoints 对象**：
   Endpoints 直接定义外部服务的 IP 或主机名。在 AWS RDS 的情况下，EndPoints 将指向 RDS 实例的主机名。

2. **创建一个 ClusterIP Service**：
   创建一个 ClusterIP 类型的 Service，让 Kubernetes 内部的应用通过该 Service 来访问外部的 RDS 数据库。

### **示例配置**

#### **1. 定义 Endpoints**

首先，创建一个 Endpoints 对象，指定 AWS RDS 实例的主机名或 IP 地址：

```yaml
apiVersion: v1
kind: Endpoints
metadata:
  name: aws-rds-endpoint
subsets:
  - addresses:
      - hostname: my-rds-instance.xxxxxxxxxx.us-east-1.rds.amazonaws.com  # RDS 主机名
    ports:
      - port: 3306  # RDS 的端口号，通常是 3306 (MySQL) 或 5432 (PostgreSQL)
```

这个 Endpoints 配置将 AWS RDS 的主机名和端口暴露给 Kubernetes 集群。

#### **2. 定义 Service**

接下来，创建一个与 Endpoints 对应的 ClusterIP 类型的 Service。Service 的作用是让 Kubernetes 内部的应用能够通过 `aws-rds-service` 名称访问外部的 RDS 数据库。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: aws-rds-service
spec:
  type: ClusterIP
  ports:
    - port: 3306  # Service 暴露的端口
      targetPort: 3306  # RDS 数据库使用的实际端口
```

在这个配置中：
- **`name: aws-rds-service`** 定义了 Service 的名称。
- **`port: 3306`** 定义了客户端通过此 Service 访问数据库时使用的端口。
- **`targetPort: 3306`** 将流量转发到 RDS 数据库的实际端口。

#### **3. 使用 Service 访问 RDS**

在 Kubernetes 集群内的应用程序可以通过 `aws-rds-service:3306` 这样的地址来访问 AWS RDS 实例，而不需要知道 RDS 的具体外部 DNS 名称。

### **使用场景**

- **简化外部服务访问**：通过这种配置，应用程序可以像访问 Kubernetes 内部的服务一样，统一通过服务名称（`aws-rds-service`）来访问外部服务。
- **集群内外一致性**：即使 RDS 的 DNS 名称或 IP 发生变化，你只需要修改 Endpoints，而不需要修改所有应用程序的配置。
- **安全性和网络策略管理**：通过 Kubernetes Service 代理外部 RDS，结合 Kubernetes 网络策略（NetworkPolicy）或 RBAC，可以对外部访问进行更好的控制。

### **总结**

通过结合 Kubernetes 的 **Service** 和 **Endpoints**，你可以将外部的 AWS RDS 数据库映射为 Kubernetes 内部的可访问资源，提供一致的服务访问方式，简化应用程序的配置管理。

## k8s 通过 service 去找到对应的endpoints的原理
```yaml
apiVersion: v1
kind: Endpoints
metadata:
#  这个名字需要和 service的名字一样才能关联上
  name: external-service
subsets:
  - addresses:
      - ip: 107.21.23.101  # 公网 IP 地址
    ports:
      - port: 80  # 假设外部资源监听的端口为 80
        protocol: TCP
---
apiVersion: v1
kind: Service
metadata:
  name: external-service
spec:
  type: NodePort
  ports:
    - port: 80        # Service 暴露的端口
      targetPort: 80  # 外部 IP 的端口（与 Endpoints 定义的端口匹配）
      protocol: TCP
```

```shell
kubectl get svc external-service 
NAME               TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
external-service   NodePort   10.100.123.45   <none>        80:30816/TCP   8m47s
bhuang@EXPJ526RGHJ02 manifests % curl localhost:30816
<h1>Welcome to StackSimplify - APP-1</h1>
```

在 Kubernetes 中，**Service** 是用于提供稳定网络入口和负载均衡的资源，它将流量转发到相应的 **Endpoints**，从而将服务请求传递到 Pod 或外部资源。Kubernetes 通过 **Service** 和 **Endpoints** 的配合，实现了高效的服务发现和流量管理。下面是 Kubernetes 如何通过 **Service** 找到对应 **Endpoints** 的原理。

### **Service 和 Endpoints 的工作原理**

1. **Service 的作用**：
   - **Service** 是 Kubernetes 中为一组 Pod 提供稳定网络入口的抽象。它允许客户端通过单一入口（如 ClusterIP、NodePort、LoadBalancer）来访问多个后端 Pod。即使 Pod 的 IP 地址发生变化，Service 依然提供稳定的访问途径。
   - 当 **Service** 关联到 Pod 时，它通常依赖于 **Label Selector** 来找到匹配的 Pod。对于外部服务，它依赖于手动定义的 **Endpoints** 对象。

2. **Endpoints 的作用**：
   - **Endpoints** 是 Kubernetes 中的另一个对象，包含了与 **Service** 相关联的 Pod 或外部服务的 IP 地址和端口信息。它用于存储 Service 将流量路由到的目标地址。对于内部 Pod，Endpoints 会自动生成并更新；对于外部服务，需要手动定义。

3. **关联方式：通过名称匹配**：
   - **Service 和 Endpoints 的名称必须完全匹配**。当 Kubernetes 创建了一个 Service 对象时，它会通过名称寻找是否有同名的 Endpoints 对象。如果找到匹配的 Endpoints，Service 将把流量路由到 Endpoints 中定义的 IP 和端口上。
   - 这种名称匹配机制确保了 Service 和 Endpoints 的绑定，并且通过 Endpoints 提供了一个对外透明的目标地址集合。

4. **Label Selector（选择器）**：
   - 对于集群内的 Pod，Service 使用 **Label Selector** 来选择与之匹配的 Pod。Service 会监视这些 Pod 的状态，当新的 Pod 启动时，它们会被自动添加到 **Endpoints** 中，而 Pod 终止时，它们会被自动移除。这个过程是动态的，Kubernetes 会持续维护 Service 和 Pod 之间的映射关系。

5. **kube-proxy 的作用**：
   - Kubernetes 中的 **kube-proxy** 是管理服务网络流量的核心组件，它负责维护服务的网络规则。当 Service 和 Endpoints 绑定后，kube-proxy 通过 **iptables** 或 **IPVS** 创建网络规则，将流量从 Service 的虚拟 IP（ClusterIP、NodePort 等）转发到 Endpoints 中的 Pod 或外部 IP。

### **通过 Service 找到 Endpoints 的步骤**

1. **Service 创建**：
   - 当创建一个 **Service** 时，Kubernetes 控制平面会根据 Service 的定义查找是否有与之关联的 **Endpoints** 对象。如果是基于 Pod 的 Service，它会通过 Label Selector 选择与之匹配的 Pod，并自动生成对应的 Endpoints 对象。

2. **动态维护 Endpoints**：
   - Kubernetes 通过监控 Pod 的生命周期，动态维护 Endpoints。如果某个 Pod 被销毁或被调度到新的节点，Kubernetes 会相应地更新 Endpoints，以确保 Service 始终指向可用的 Pod。

3. **流量转发**：
   - 当客户端请求到达 **Service** 时，kube-proxy 会根据已配置的网络规则将请求转发到对应的 **Endpoints**。如果 Endpoints 中包含多个 Pod 的 IP 地址，kube-proxy 会实现负载均衡，将流量分发到这些 Pod。

4. **外部服务场景**：
   - 当你要通过 Kubernetes Service 代理外部服务时，可以手动定义 Endpoints，指定外部服务的 IP 地址和端口。Service 会通过与这些手动定义的 Endpoints 建立映射，从而将流量转发到外部的服务。

### **总结**

Kubernetes 中的 **Service** 通过与 **Endpoints** 的绑定，将流量路由到具体的 Pod 或外部资源。其核心原理是通过名称匹配的方式来找到相应的 Endpoints 对象，并利用 **kube-proxy** 维护的网络规则实现流量的转发。对于集群内部的 Pod，Service 通过 Label Selector 动态管理 Pod 与 Endpoints 的关系；对于外部服务，手动创建的 Endpoints 充当目标资源的地址集合。

这种机制使得 Kubernetes 在提供高可用性、服务发现和负载均衡方面具有很强的扩展性和灵活性【30†source】【32†source】.

### k8s 内 的service 访问 具有公网的 ip 的资源

## ExternalName 代理 域名

## 多端口 service

## 什么是 Ingress (Ingress 是 配置， Ingress Controller 是服务)
```md
Ingress 类比 成 nginx.conf, 而Ingress Controller可以类比成是Nginx
```

在 Kubernetes 中，**Ingress** 是一种 API 对象，它允许你管理集群内部的 HTTP 和 HTTPS 路由，提供外部访问集群服务的途径。通过 Ingress，你可以定义和控制如何从外部访问 Kubernetes 集群中的服务，并且可以根据请求路径或主机名将流量路由到不同的服务。

### **Ingress 的功能**

1. **HTTP/HTTPS 路由**：
   - Ingress 允许你根据请求的 **URL 路径** 和 **主机名** 来定义规则，将流量路由到相应的 Kubernetes 服务。例如，你可以设置 `/app1` 路径的请求流量转发到 `app1-service`，而 `/app2` 的请求转发到 `app2-service`。

2. **TLS（HTTPS 支持）**：
   - Ingress 支持 **TLS** 配置，可以为服务提供 HTTPS 访问。你可以为不同的域名配置不同的 TLS 证书，使外部用户可以通过加密的方式访问服务。

3. **反向代理和负载均衡**：
   - Ingress 通常结合 **Ingress Controller** 来实现反向代理和负载均衡的功能。流量首先进入 Ingress Controller，它根据 Ingress 规则将流量路由到对应的服务，实现反向代理和负载均衡。

4. **虚拟主机**：
   - Ingress 允许根据请求的主机名（如 `example.com` 或 `app.example.com`）将流量转发到不同的服务，支持基于主机名的虚拟主机功能。

### **Ingress 的使用场景**

1. **HTTP 和 HTTPS 负载均衡**：
   - 当你需要对多个 Kubernetes 服务进行 HTTP 或 HTTPS 的负载均衡时，Ingress 是非常合适的解决方案。它提供了集中的入口，并且可以根据请求的路径或域名路由流量。

2. **多域名和路径支持**：
   - Ingress 可以根据域名或路径来灵活路由流量，支持不同的子域名和路径映射到不同的 Kubernetes 服务，特别适合有多个微服务的系统。

3. **安全访问**：
   - Ingress 支持 TLS，可以为服务提供 HTTPS 安全访问。你可以为每个域名配置单独的 TLS 证书，确保外部访问的安全性。

### **Ingress 的架构与组件**

1. **Ingress Resource（Ingress 资源）**：
   - Ingress 是 Kubernetes 中的 API 对象，它定义了 HTTP 和 HTTPS 访问集群内服务的规则。这些规则包括主机名、路径、目标服务等。

2. **Ingress Controller**：
   - Ingress 资源本身只定义了规则，实际的流量路由由 **Ingress Controller** 负责处理。Ingress Controller 是部署在 Kubernetes 集群中的控制器，它会监视 Ingress 资源，并根据规则配置相应的路由。常见的 Ingress Controller 实现有 **NGINX Ingress Controller**、**Traefik**、**HAProxy** 等。

3. **Service 和 Pod 的连接**：
   - Ingress 并不直接与 Pod 通信，而是通过 Kubernetes 的 **Service** 对象将流量转发到 Pod。因此，Ingress 是基于 Service 的，流量最终会被路由到与 Service 关联的 Pod 上。

### **示例：定义 Ingress 规则**

下面是一个简单的 Ingress 配置示例，它展示了如何根据路径和域名将流量转发到不同的服务：

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example-ingress
spec:
  rules:
  - host: example.com
    http:
      paths:
      - path: /app1
        pathType: Prefix
        backend:
          service:
            name: app1-service
            port:
              number: 80
      - path: /app2
        pathType: Prefix
        backend:
          service:
            name: app2-service
            port:
              number: 80
  tls:
  - hosts:
    - example.com
    secretName: example-tls
```

在这个例子中：
- **`host: example.com`**：定义了流量基于 `example.com` 域名。
- **路径 `/app1` 和 `/app2`**：分别将流量路由到 `app1-service` 和 `app2-service`。
- **TLS**：配置了 `example-tls` 证书，用于为 `example.com` 提供 HTTPS 支持。

### **Ingress Controller 的重要性**

**Ingress Controller** 是 Ingress 正常运行的关键，它负责解析 Ingress 资源中的规则并将其转换为实际的网络配置。常见的 Ingress Controller 包括：
- **NGINX Ingress Controller**：最常见的 Ingress Controller 之一，基于 NGINX 实现反向代理和负载均衡。
- **Traefik**：也是一个流行的 Ingress Controller，提供简单的配置和高级功能，如动态证书管理。

### **总结**

- **Ingress** 是 Kubernetes 中管理 HTTP 和 HTTPS 流量的资源，它为外部访问集群内部服务提供灵活的路由规则和负载均衡。
- **Ingress Controller** 是实现 Ingress 流量路由的组件，常见的实现有 NGINX、Traefik 等。
- Ingress 支持通过域名、路径规则将流量转发到不同的服务，并支持 HTTPS/TLS 安全访问.

## 使用Ingress 发布服务的流程

## Ingress Controller 安装
- https://kubernetes.github.io/ingress-nginx/deploy/#quick-start



## 使用域名发布k8s 内部服务
要在 Docker Hub 启动的 Kubernetes 集群中配置 Ingress 访问服务，需要按照以下步骤进行设置。你需要创建一个 Ingress Controller（例如 NGINX Ingress Controller），然后定义 Ingress 资源来将流量路由到你的 Kubernetes 服务。

### **步骤 1：安装 Ingress Controller**

在 Kubernetes 中，**Ingress Controller** 是处理和路由 Ingress 流量的关键组件。这里使用 **NGINX Ingress Controller** 作为示例：

1. **安装 NGINX Ingress Controller**：
   可以使用 `kubectl` 命令安装 NGINX Ingress Controller。如果你正在运行 Kubernetes，并且没有设置外部负载均衡器（如云提供商提供的负载均衡器），建议通过 **NodePort** 暴露 Ingress Controller。

   使用以下命令通过 **Helm** 安装 NGINX Ingress Controller：

   ```bash
   helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
   helm repo update
   helm install my-ingress ingress-nginx/ingress-nginx
   ```

   或者，直接应用 NGINX Ingress Controller 的 YAML 文件：

   ```bash
   kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml
   ```

   这将安装 NGINX Ingress Controller，并且会在集群中启动相关的 Pod。

2. **验证 Ingress Controller 安装**：
   确认 NGINX Ingress Controller 正在运行：

   ```bash
   kubectl get pods -n ingress-nginx
   ```

   你应该能够看到与 `ingress-nginx` 相关的 Pod 正在运行。

### **步骤 2：创建 Kubernetes 服务**

在你的 Kubernetes 集群中，你可能已经有了一个要通过 Ingress 公开的服务。如果没有，首先需要创建一个 Service。

示例：假设你有一个 NGINX 服务运行在集群中：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: ClusterIP
```

应用此 YAML 文件：

```bash
kubectl apply -f nginx-service.yaml
```

这将创建一个名为 `nginx-service` 的服务，运行在集群内部。

### **步骤 3：创建 Ingress 资源**

现在，创建 Ingress 资源，将外部流量引导到 Kubernetes 服务。

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: nginx.example.com  # 用于外部访问的域名
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-service  # Kubernetes 中的服务名称
            port:
              number: 80
```

这个 Ingress 规则表示当请求到达 `nginx.example.com` 时，流量会被路由到 `nginx-service` 服务的端口 80。

应用 Ingress 资源：

```bash
kubectl apply -f nginx-ingress.yaml
```

### **步骤 4：配置 DNS（或本地 hosts 文件）**

为了测试 Ingress 的功能，你可以在本地机器的 `/etc/hosts` 文件中添加 DNS 映射，以便将 `nginx.example.com` 指向 Ingress Controller 暴露的 IP 地址。

1. 查找 Ingress Controller 暴露的 IP 地址：

   ```bash
   kubectl get svc -n ingress-nginx
   ```

   找到 `ingress-nginx-controller` 服务的 `EXTERNAL-IP` 或者 `NodePort`。
![img_4.png](..%2Fimg%2Fimg_4.png)

2. 修改 `/etc/hosts` 文件，将 `nginx.example.com` 指向该 IP 地址：

   ```bash
   sudo nano /etc/hosts
   ```

   添加一行：
![img_5.png](..%2Fimg%2Fimg_5.png)
   ```
   这里Ingress-Controller-IP 就是localhost, 因为我在dockerhub启动的k8s
   <Ingress-Controller-IP> nginx.example.com
   ```

### **步骤 5：测试 Ingress 访问**

现在，你可以通过 `curl` 或浏览器访问 `nginx.example.com` 来测试是否能够通过 Ingress 访问 NGINX 服务：

```bash
curl http://nginx.example.com:31838
```
![img_6.png](..%2Fimg%2Fimg_6.png)

### **总结**

- **Ingress Controller** 是必须的，它负责实际的流量路由。
- 你需要创建一个 Ingress 资源，通过规则（如路径或域名）将流量引导到 Kubernetes 服务。
- 配置 DNS 或 `/etc/hosts` 文件，将域名指向 Ingress Controller 的 IP 地址。

## 将 nginx.example.com 指向该 IP 地址 ,Ingress-Controller-IP 是指？
**Ingress-Controller-IP** 指的是 **Ingress Controller** 服务的外部 IP 地址，也就是当你部署了 Kubernetes 的 Ingress Controller 后，用来处理外部流量的负载均衡器或节点的 IP 地址。

在你本地测试环境中（例如使用 **Docker Desktop** 或 **Minikube**），如果没有自动分配真正的外部 IP，通常会显示为 **localhost** 或 **127.0.0.1**。这意味着流量会通过本地机器处理，而不需要真正的外部云负载均衡器。

### **步骤：修改 `/etc/hosts` 文件**

1. **确定 Ingress Controller 的 IP**：
   - 运行以下命令，获取 Ingress Controller 的外部 IP 地址：
     ```bash
     kubectl get svc -n ingress-nginx
     ```

   如果 **EXTERNAL-IP** 显示为 `localhost`，这意味着你的 Ingress Controller 没有被分配外部 IP，表示本地机器将处理流量。如果它显示一个实际的 IP 地址（如在云环境中），则用该 IP。

   示例输出：
   ```
   NAME                     TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)                      AGE
   ingress-nginx-controller  LoadBalancer   10.101.222.216   203.0.113.10   80:31838/TCP,443:32086/TCP   6h5m
   ```

   在此示例中，**Ingress-Controller-IP** 是 `203.0.113.10`。

2. **编辑 `/etc/hosts` 文件**：
   - 打开并编辑本地的 `/etc/hosts` 文件（在 Linux 或 macOS 上）：
     ```bash
     sudo nano /etc/hosts
     ```

   - 如果你是在 Windows 系统中，`hosts` 文件位于 `C:\Windows\System32\drivers\etc\hosts`。

3. **添加 Ingress Controller IP 和域名映射**：
   - 添加一行，将域名映射到 **Ingress-Controller-IP**。例如：
     ```
     203.0.113.10  nginx.example.com
     ```

   - 如果你的 **EXTERNAL-IP** 是 `localhost`，可以将其映射为：
     ```
     127.0.0.1  nginx.example.com
     ```

4. **保存并退出**：
   - 保存文件后，DNS 解析会将 `nginx.example.com` 映射到你指定的 IP 地址。

5. **访问 Ingress 服务**：
   - 现在，你可以通过 `curl` 或浏览器访问 `nginx.example.com`，无需指定端口号：
     ```bash
     curl http://nginx.example.com
     ```

### **总结**

- **Ingress-Controller-IP** 是 Ingress Controller 的外部 IP 地址，通常是由 **LoadBalancer** 类型的服务分配的。如果是在本地环境，可能会是 `localhost`。
- 修改 `/etc/hosts` 文件后，浏览器或命令行工具将会解析 `nginx.example.com` 为你指定的 IP，从而访问 Kubernetes 中的 Ingress Controller 服务。



## Ingress 特例： 不配置域名发布服务

## Ingress v1 和 v1beta1 的区别

