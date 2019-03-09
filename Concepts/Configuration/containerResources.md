### 管理容器计算资源

当你配置一个pod时，可以选择指定每个容器需要的CPU和内存（RAM）。当容器指定了requests资源时，调度器可以更好的决定将pod部署到哪个节点上。当容器指定了limits资源时，可以以指定的方式处理节点上资源的争用。requests和limits之间区别更详细的信息可以从[Resource QoS](https://git.k8s.io/community/contributors/design-proposals/node/resource-qos.md)获取。本文将针对以下主题分别介绍：

*   资源类型

*   Resource requests and limits of Pod and Container

*   Meaning of CPU

*   [Meaning of memory](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/#meaning-of-memory)

*   [How Pods with resource requests are scheduled](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/#how-pods-with-resource-requests-are-scheduled)

*   [How Pods with resource limits are run](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/#how-pods-with-resource-limits-are-run)

*   [Monitoring compute resource usage](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/#monitoring-compute-resource-usage)

*   [Troubleshooting](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/#troubleshooting)

*   [Local ephemeral storage](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/#local-ephemeral-storage)

*   [Extended resources](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/#extended-resources)

*   [Planned Improvements](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/#planned-improvements)

### 资源类型

*CPU*和*内存*都是资源类型，每种资源都有基本单位，CPU的基本单位是核，内存的基本单位是字节。CPU和内存统称为计算资源，或仅称资源。计算资源是可以请求，分配和使用的可测量数量，不同与API 资源。API 资源（Pods和Services）是可以通过Kubernetes API Server可以创建和修改的对象。

### Pod、容器资源：requests和limits

Pod的每个Container都可以指定以下一项或多项：

*   `spec.containers[].resources.limits.cpu`

*   `spec.containers[].resources.limits.memory`

*   `spec.containers[].resources.requests.cpu`

*   `spec.containers[].resources.requests.memory`

尽管requests和limits只能对单个容器进行配置，但是讨论Pod资源的requests和limits也是十分方便的。一个Pod对某种特定资源的request/limit是Pod内所有容器request/limit的总和。

### CPU的意义

CPU资源的Limits和requests的值都以*cpu*单元进行计算。在Kubernetes中，一cpu和以下cpu的表示对等：

- 1 AWS vCPU
- 1 GCP Core
- 1 Azure vCore
- 1 IBM vCPU
- 一个超线程（在具有超线程的裸机Intel处理器上）

CPU资源允许请求分数个cpu。一个拥有`spec.containers[].resources.requests.cpu=0.5`的容器保证容器之多请求0.5个CPU。一个CPU=1000m,对于CPU资源的精度为1m，精度超过1m的设置是不允许的。CPU资源都是以绝对值而不是相对值申请，在一个只有一核，双核或者48核的机器上，0.1的含义是一样的。

### 内存的意义

内存资源的Limit和request值都是以字节为单位进行计算。可以使用普通整数或定点整数配合后缀 E, P, T, G, M, K来表示内存，当然也可以使用两个字母的后缀Ei, Pi, Ti, Gi, Mi, Ki。举个例子，以下表达式计算的内存值是相等的：

```shell
128974848, 129e6, 129M, 123Mi
```

下面有一个简单的例子，一个Pod有两个容器，每个容器需要0.25CPU和64MiB内存，每个容器限制最多使用0.5CPU和128MiB内存。可以说整个Pod需要0.5CPU和128MiB内存，最多使用1CPU和256MiB内存。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: frontend
spec:
  containers:
  - name: db
    image: mysql
    env:
    - name: MYSQL_ROOT_PASSWORD
      value: "password"
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
  - name: wp
    image: wordpress
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
```

### 如何调度对资源requests的pod

When you create a Pod, the Kubernetes scheduler selects a node for the Pod to run on. Each node has a maximum capacity for each of the resource types: the amount of CPU and memory it can provide for Pods. The scheduler ensures that, for each resource type, the sum of the resource requests of the scheduled Containers is less than the capacity of the node. Note that although actual memory or CPU resource usage on nodes is very low, the scheduler still refuses to place a Pod on a node if the capacity check fails. This protects against a resource shortage on a node when resource usage later increases, for example, during a daily peak in request rate.

### How Pods with resource limits are run