# 资源配额

当多个用户或团队共享具有固定数量节点的群集时，有人担心一个团队可能会使用超过其公平份额的资源。资源配额是管理员解决此问题的工具。本章包括以下内容：

*   启用资源配额
*   计算资源配额
*   存储资源配额
*   对象数配额
*   配额范围
*   Requests vs Limits
*   配额呈现和设置
*   配额和集群容量
*   Priority Class默认消费限制

由ResourceQuota对象定义的资源配额提供限制每个命名空间的聚合资源消耗的约束。它可以按类型限制可在命名空间中创建的对象数量，以及该项目中资源可能消耗的计算资源总量。

资源配额的工作方式如下：

*   不同的团队在不同的命名空间中工作。目前这是自愿的，但计划通过ACL实现强制性支持。

*   管理员为每个命名空间创建一个`ResourceQuota`

*   用户在命名空间中创建资源（窗格，服务等），配额系统跟踪使用情况以确保它不超过ResourceQuota中定义的硬资源限制。

*   如果创建或更新资源违反配额约束，则请求将失败，HTTP状态代码403 FORBIDDEN，并显示一条消息，说明可能违反的约束。

*   如果在命名空间内启用了对计算资源（cpu和内存）的配额，用户必须对cpu和内存设置需要和限制值；不然，配额系统可能拒绝pod创建。提示：使用LimitRanger控制器强制不需要计算资源的pod的默认值。

命名空间和配额创建可以使用的策略示例如下：

*   在一个拥有16核，32GB内粗的集群内。设置团队A使用20GB内存和10个CPU核，团队B使用10GB内部和4个CPU核，余下的2GB内存和2核用于将来分配使用。
*   限制testing命名空间使用1核1GB内存，production命名空间使用任意多资源。

在集群的总容量小于命名空间的配额总和的情况下，可能存在资源争用。 这是以先到先得的方式处理的。

争用和配额更改都不会影响已创建的资源。



### 启用资源配额

------

许多Kubernetes的发行版本默认支持资源配额，当apiserver 的**--enable-admission-plugins=**标签以**ResourceQuota **作为一个参数时，kubernetes开启资源配额。

当某个命名空间中存在**ResourceQuota **时，该命名空间强制开启资源配额。



### 计算资源配额

------

您可以限制给定命名空间中可请求的计算资源的总和。Kubernetes支持以下资源类型：

| 资源名            | 描述                                                |
| ----------------- | --------------------------------------------------- |
| `cpu`             | 所有非停止状态的Pod中，请求的CPU总和不能超过此值。  |
| `limits.cpu`      | 所有非停止状态的Pod中，限制的CPU总和不能超过此值。  |
| `limits.memory`   | 所有非停止状态的Pod中，限制的内存总和不能超过此值。 |
| `memory`          | 所有非停止状态的Pod中，内存请求的总和不能超过此值。 |
| `requests.cpu`    | 所有非停止状态的Pod中，请求的CPU总和不能超过此值。  |
| `requests.memory` | 所有非停止状态的Pod中，请求的内存总和不能超过此值。 |

#### 资源配额：扩展资源

除去上面提到的资源类型，在发行的1.10版本中，增加了配额对扩展资源的支持。

由于不允许对扩展资源进行过度使用，因此在配额中为同一扩展资源指定请求和限制是没有意义的。因此对于扩展资源，当前仅支持前缀为**requests.** 的配额。

以GPU资源为例，如果资源的名字为**nvidia.com/gpu**，如果你想限制命名空间内对GPU资源的请求上线为4，你可以使用如下的方式进行定义：

- `requests.nvidia.com/gpu: 4`



### 存储资源配额

------

您可以限制给定命名空间中可以请求的存储资源的总和。此外，您可以根据关联的存储类限制存储资源的使用。

| 资源名                                                       | 描述                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| `requests.storage`                                           | 在所有持久性卷声明中，存储请求的总和不能超过此值。           |
| `persistentvolumeclaims`                                     | 命名空间中可以存在的持久性卷声明的总数。                     |
| `<storage-class-name>.storageclass.storage.k8s.io/requests.storage | 在与存储类名称关联的所有持久性卷声明中，存储请求的总和不能超过此值。 |
| `<storage-class-name>.storageclass.storage.k8s.io/persistentvolumeclaims` | 在与存储类名相关联的所有持久性卷声明中，命名空间中可以存在的持久性卷声明的总数。 |

例如，如果运营商想要使用与青铜存储类别分开的黄金存储类配额存储，则运营商可以按如下方式定义配额：

- `gold.storageclass.storage.k8s.io/requests.storage: 500Gi`
- `bronze.storageclass.storage.k8s.io/requests.storage: 100Gi`

In release 1.8, quota support for local ephemeral storage is added as an alpha feature:

在1.8版中，添加了对本地临时存储的配额支持作为alpha功能：

| 资源名                       | 描述                                                         |
| ---------------------------- | ------------------------------------------------------------ |
| `requests.ephemeral-storage` | 在命名空间中的所有pod中，本地临时存储请求的总和不能超过此值。 |
| `limits.ephemeral-storage`   | 在命名空间中的所有pod中，本地临时存储限制的总和不能超过此值。 |

### 对象数配额

------

1.9版本使用以下语法添加了对所有标准命名空间资源类型的配额支持：

- `count/<resource>.<group>`

以下是用户可能希望置于对象计数配额下的一组示例资源：

- `count/persistentvolumeclaims`
- `count/services`
- `count/secrets`
- `count/configmaps`
- `count/replicationcontrollers`
- `count/deployments.apps`
- `count/replicasets.apps`
- `count/statefulsets.apps`
- `count/jobs.batch`
- `count/cronjobs.batch`
- `count/deployments.extensions`

当使用**count / ***资源配额时，如果对象存在于服务器存储中，则会根据配额收费。这些类型的配额可用于防止存储资源耗尽。例如，你可能想对服务器上存储的密钥数量进行配额管理。集群存储太多的密钥可能会阻止服务器和控制器的启动。您可以选择**Job**配额以防止配置不当的**cronjob**在命名空间中创建过多的**Job**，从而导致拒绝服务。

在1.9版本之前，可以在有限的资源集上执行通用对象计数配额。 此外，还可以通过其类型进一步约束特定资源的配额。

支持以下类型:

| 资源名                   | 描述                                              |
| ------------------------ | ------------------------------------------------- |
| `configmaps`             | 命名空间内可以存在的**configmaps**总数。          |
| `persistentvolumeclaims` | 命名空间内可以存在的persistentvolumeclaim总数。   |
| `pods`                   | 命名空间内非停止状态的pod的总数。                 |
| `replicationcontrollers` | 命名空间内可以存在的replicationcontroller总数。   |
| `resourcequotas`         | 命名空间内可以存在的resourcequota的总数。         |
| `services`               | 命名空间内可以存在的service总数                   |
| `services.loadbalancers` | 命名空间内可以存在的services.loadbalancer的总数。 |
| `services.nodeports`     | 命名空间内可以存在的node port的总数。             |
| `secrets`                | 命名空间内可以存在的secret的总数。                |

例如，**pods** 配额会计算并强制在单个命名空间中创建的不是终端pod数量的最大值。您可能希望在命名空间上设置pods配额，以避免用户创建许多小pod并耗尽群集的Pod IP。

### 配额范围

------

每个配额都有一个与之关联的范围集合。如果资源与枚举范围的交集匹配，则配额测量资源的使用情况，否则将不受配额的限制。

将范围添加到配额时，会将其支持的资源数限制为与范围相关的资源。在允许集合之外的配额上指定的资源会导致验证错误。

| 范围             | Description                                       |
| ---------------- | ------------------------------------------------- |
| `Terminating`    | 与 `.spec.activeDeadlineSeconds >= 0`的pod匹配    |
| `NotTerminating` | 与 `.spec.activeDeadlineSeconds is nil` 的pod匹配 |
| `BestEffort`     | 匹配具有尽力服务质量的pod。                       |
| `NotBestEffort`  | 匹配没有尽力服务质量的pod。                       |

`BestEffort`范围限制配额跟踪以下资源：`pods`

`Terminating`, `NotTerminating`, and `NotBestEffort`范围限制资源配额跟踪以下资源：

- `cpu`
- `limits.cpu`
- `limits.memory`
- `memory`
- `pods`
- `requests.cpu`
- `requests.memory`

#### 每个资源配额的PriorityClass

**特性状态:** `Kubernetes 1.12` [beta](https://kubernetes.io/docs/concepts/policy/resource-quotas/#)

可以以特定的优先级创建pod。通过quota spec中的`scopeSelector`字段，您可以根据pod的优先级控制pod对系统资源的消耗。

仅当quota spec中的`scopeSelector`选择pod时，才会匹配和使用配额。

> **注意**：您需要在每个PriorityClass使用资源配额之前启用功能`ResourceQuotaScopeSelectors`

此示例创建一个配额对象，并将其与特定优先级的pod进行匹配。 该示例的工作原理如下：

- 集群中的pod的优先级为 “low”, “medium”, “high”之一。
- 为每一种优先级创建一个资源配额。

将以下内容保存到 `quota.yml`文件中：

~~~yaml
apiVersion: v1
kind: List
items:
- apiVersion: v1
  kind: ResourceQuota
  metadata:
    name: pods-high
  spec:
    hard:
      cpu: "1000"
      memory: 200Gi
      pods: "10"
    scopeSelector:
      matchExpressions:
      - operator : In
        scopeName: PriorityClass
        values: ["high"]
- apiVersion: v1
  kind: ResourceQuota
  metadata:
    name: pods-medium
  spec:
    hard:
      cpu: "10"
      memory: 20Gi
      pods: "10"
    scopeSelector:
      matchExpressions:
      - operator : In
        scopeName: PriorityClass
        values: ["medium"]
- apiVersion: v1
  kind: ResourceQuota
  metadata:
    name: pods-low
  spec:
    hard:
      cpu: "5"
      memory: 10Gi
      pods: "10"
    scopeSelector:
      matchExpressions:
      - operator : In
        scopeName: PriorityClass
        values: ["low"]
~~~

使用 `kubectl create`将上述文件应用到集群中。

```shell
kubectl create -f ./quota.yml
# 命令的输出如下
resourcequota/pods-high created
resourcequota/pods-medium created
resourcequota/pods-low created
```

 使用`kubectl describe quota`可以查看配额的使用情况（当前使用量为0）：

```shell
kubectl describe quota
Name:       pods-high
Namespace:  default
Resource    Used  Hard
--------    ----  ----
cpu         0     1k
memory      0     200Gi
pods        0     10


Name:       pods-low
Namespace:  default
Resource    Used  Hard
--------    ----  ----
cpu         0     5
memory      0     10Gi
pods        0     10


Name:       pods-medium
Namespace:  default
Resource    Used  Hard
--------    ----  ----
cpu         0     10
memory      0     20Gi
pods        0     10
```

创建具有高优先级的Pod. 将如下内容保存到 `high-priority-pod.yml`文件中。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: high-priority
spec:
  containers:
  - name: high-priority
    image: ubuntu
    command: ["/bin/sh"]
    args: ["-c", "while true; do echo hello; sleep 10;done"]
    resources:
      requests:
        memory: "10Gi"
        cpu: "500m"
      limits:
        memory: "10Gi"
        cpu: "500m"
  priorityClassName: high
```

使用 `kubectl create`创建Pod：

```shell
kubectl create -f ./high-priority-pod.yml
```

确认高优先级配额的使用使用已经发生变化，但是其他两种配额的使用没有变化：

```shell
kubectl describe quota
Name:       pods-high
Namespace:  default
Resource    Used  Hard
--------    ----  ----
cpu         500m  1k
memory      10Gi  200Gi
pods        1     10


Name:       pods-low
Namespace:  default
Resource    Used  Hard
--------    ----  ----
cpu         0     5
memory      0     10Gi
pods        0     10


Name:       pods-medium
Namespace:  default
Resource    Used  Hard
--------    ----  ----
cpu         0     10
memory      0     20Gi
pods        0     10
```

`scopeSelector` 支持以下**操作符** `:

- `In`
- `NotIn`
- `Exist`
- `DoesNotExist`

### Requests vs Limits

------

在分配计算资源时，每个容器可以为CPU或内存指定request和limit值。 配额为CPU和内存任一配置值。如果配额的`requests.cpu`或`requests.memory`设置了值，则每一个待创建的容器需要显示的请求这些计算资源。如果配额的 `limits.cpu` 或 `limits.memory`设置了值，配额系统要求每个待创建的容器对计算资源设置一个限制。

### 配额呈现和设置

------

Kubectl支持创建、更新和查看配额：

```shell
# 创建命名空间
kubectl create namespace myspace
# 创建计算资源配额yaml文件
cat <<EOF > compute-resources.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-resources
spec:
  hard:
    pods: "4"
    requests.cpu: "1"
    requests.memory: 1Gi
    limits.cpu: "2"
    limits.memory: 2Gi
    requests.nvidia.com/gpu: 4
EOF
# 创建资源配额
kubectl create -f ./compute-resources.yaml --namespace=myspace
# 创建对象配额yaml文件
cat <<EOF > object-counts.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: object-counts
spec:
  hard:
    configmaps: "10"
    persistentvolumeclaims: "4"
    replicationcontrollers: "20"
    secrets: "10"
    services: "10"
    services.loadbalancers: "2"
EOF
# 创建对象配额
kubectl create -f ./object-counts.yaml --namespace=myspace
# 获取指定命名空间内的所有配额
kubectl get quota --namespace=myspace
NAME                    AGE
compute-resources       30s
object-counts           32s

# 查看 compute-resources配额详情
kubectl describe quota compute-resources --namespace=myspace
Name:                    compute-resources
Namespace:               myspace
Resource                 Used  Hard
--------                 ----  ----
limits.cpu               0     2
limits.memory            0     2Gi
pods                     0     4
requests.cpu             0     1
requests.memory          0     1Gi
requests.nvidia.com/gpu  0     4
# 查看object-counts配额详情
kubectl describe quota object-counts --namespace=myspace
Name:                   object-counts
Namespace:              myspace
Resource                Used    Hard
--------                ----    ----
configmaps              0       10
persistentvolumeclaims  0       4
replicationcontrollers  0       20
secrets                 1       10
services                0       10
services.loadbalancers  0       2
```

Kubectl使用`count/<resource>.<group>`语法支持对命名空间内标准资源创建数量配额：

```shell
kubectl create namespace myspace
kubectl create quota test --hard=count/deployments.extensions=2,count/replicasets.extensions=4,count/pods=3,count/secrets=4 --namespace=myspace
kubectl run nginx --image=nginx --replicas=2 --namespace=myspace
kubectl describe quota --namespace=myspace
Name:                         test
Namespace:                    myspace
Resource                      Used  Hard
--------                      ----  ----
count/deployments.extensions  1     2
count/pods                    2     3
count/replicasets.extensions  1     4
count/secrets                 1     4
```

### 配额和集群容量

------

`ResourceQuotas` 独立于集群容量，它们都以绝对单位表示。因此，如果向集群添加节点，每个命名空间并没有能力自动消费更多的资源。

有时可能需要更复杂的策略，比如：

- 在多个团队内按比例划分集群资源。
- 允许每个租户根据需要增加资源使用，但有一个适度的限制，以防止意外资源耗尽。
- 检测一个命名空间的需求，添加节点并增加配额.

这些策略可以使用ResourceQuotas作为构建块来实现，方法是编写一个监视配额使用情况的“控制器”，并根据其他信号调整每个命名空间的配额硬限制。请注意，资源配额会对聚合群集资源进行划分，但它不会对节点产生任何限制：来自多个命名空间的pod可能在同一节点上运行。

### 默认限制优先级

------

It may be desired that pods at a particular priority, eg. “cluster-services”, should be allowed in a namespace, if and only if, a matching quota object exists.

可能希望具有特定优先级的容器，例如“cluster-services”，当且仅当存在匹配的配额对象时，才允许在命名空间中存在。使用此机制，运算符将能够将某些高优先级类的使用限制为有限数量的命名空间，并且默认情况下并非每个命名空间都能够使用这些优先级类。为了启用此功能，下面的配置需要作为kube-apiserver的 `--admission-control-config-file` 参数传入：

```yaml
apiVersion: apiserver.k8s.io/v1alpha1
kind: AdmissionConfiguration
plugins:
- name: "ResourceQuota"
  configuration:
    apiVersion: resourcequota.admission.k8s.io/v1beta1
    kind: Configuration
    limitedResources:
    - resource: pods
      matchScopes:
      - scopeName: PriorityClass 
        operator: In
        values: ["cluster-services"]
```

现在，只有那些存在匹配`scopeSelector`的配额对象的命名空间中才允许使用“集群服务”窗格。 例如：

```yaml
    scopeSelector:
      matchExpressions:
      - scopeName: PriorityClass
        operator: In
        values: ["cluster-services"]
```



