### 管理容器计算资源

当你配置一个pod时，可以选择指定每个容器需要的CPU和内存（RAM）。当容器指定了requests资源时，调度器可以更好的决定将pod部署到哪个节点上。当容器指定了limits资源时，可以以指定的方式处理节点上资源的争用。requests和limits之间区别更详细的信息可以从[Resource QoS](https://git.k8s.io/community/contributors/design-proposals/node/resource-qos.md)获取。本文将针对以下主题分别介绍：

*   资源类型

*   Pod、容器资源request和limit

*   CPU的意义

*   内存的意义

*   如何调度Pod

*   如何运行Pod

*   监控计算资源使用

*   故障排除

*   本地短暂存储

*   扩展资源

*   改进计划

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

### 如何调度对资源有要求的pod

------

当你创建一个pod时，kubernete的调度器会选择一个节点允许pod。节点上每种资源都有一个最大容量：可以为pod提供的CPU和内存的总和。调度器保证，对于每种资源，调度容器需要的总和低于节点的容量。注意，即使节点cpu和内存的使用非常低，如果容量检查失败，调度器依然会拒绝讲pod调度到该节点。当稍后资源的使用增加时，这可以有效的阻止节点上资源的短缺，例如，在一天中请求最高的时刻。

### 如何允许使用资源有限制的pod

------

当kubelet启动一个容器pod，kubelet把CPU和内存的限制传给容器运行时。

当使用Docker时:

- ~~spec.containers[].resources.requests.cpu的值会转换为核数，如果值为分数则乘1024，并将该值作为docker run --cpu-shares的值。~~
-  `spec.containers[].resources.limits.cpu` 值转换为微核值并乘100，该值为容器每100ms内使用的总的cpu事件。在100ms的时间间隔内容器不可以使用超过容器共享的cpu时间。

> **注意:** 默认的配额周期为100ms，cpu分配时间的周期是1ms。

- `spec.containers[].resources.limits.memory` 值转换为一个整数，并作为`docker run --memory`的值用于启动容器。

如果一个容器使用了超过内存限制的内存，容器可能被停止。如果容器可以重启，kubelet会重启容器，其他类型的运行时错误也一样。如果一个容器超过了对内存的需要值，当节点内存不足时，pod可能被驱逐。容器长时间使用超过cpu限制可能会被杀死也可能不会被杀死。但是，容器不会因为cpu使用率高而被杀死。

### 监控计算资源的使用

------

资源使用是pod状态报告的一部分。如果kubernetes集群配置了可选的[监控](https://github.com/kubernetes/kubernetes/blob/master/cluster/addons/cluster-monitoring/README.md)插件，pod对资源的使用可以从监控系统获取。

### 故障排除

------

#### Pod一直处于pending状态，错误信息为调度失败

如果调度器找不到任何一个可以允许pod的节点，在找到这样的节点之前pod一直处于未调度状态。每次调度器查找节点失败时，会产生如下类似的一个事件：

```shell
kubectl describe pod frontend | grep -A 3 Events
Events:
  FirstSeen LastSeen   Count  From          Subobject   PathReason      Message
  36s   5s     6      {scheduler }              FailedScheduling  Failed for reason PodExceedsFreeCPU and possibly others
```

在上面的例子中，一个名叫“frontend”的pod因为节点的cpu资源不足而调度失败，当节点内存不足时会出现类似的错误信息（PodExceedsFreeMemory）。一般来说，如果pod处于pending状态并且错误信息未cpu资源不足或内存资源不足时，可以尝试以下方法解决：

- 向集群添加更多的节点。
- 停止不需要的pod，将资源提供给pending状态的pods。
- 检查pod不大于所有节点。例如，所有节点的cpu容量为1，但是一个pod对cpu的需求为1.1，则该pod永远无法被调度。

使用 kubectl describe node 可以查看节点所有资源和已经分配的资源。

```shell
kubectl describe nodes e2e-test-minion-group-4lw4
Name:            e2e-test-minion-group-4lw4
[ ... lines removed for clarity ...]
Capacity:
 cpu:                               2
 memory:                            7679792Ki
 pods:                              110
Allocatable:
 cpu:                               1800m
 memory:                            7474992Ki
 pods:                              110
[ ... lines removed for clarity ...]
Non-terminated Pods:        (5 in total)
  Namespace    Name                                  CPU Requests  CPU Limits  Memory Requests  Memory Limits
  ---------    ----                                  ------------  ----------  ---------------  -------------
  kube-system  fluentd-gcp-v1.38-28bv1               100m (5%)     0 (0%)      200Mi (2%)       200Mi (2%)
  kube-system  kube-dns-3297075139-61lj3             260m (13%)    0 (0%)      100Mi (1%)       170Mi (2%)
  kube-system  kube-proxy-e2e-test-...               100m (5%)     0 (0%)      0 (0%)           0 (0%)
  kube-system  monitoring-influxdb-grafana-v4-z1m12  200m (10%)    200m (10%)  600Mi (8%)       600Mi (8%)
  kube-system  node-problem-detector-v0.1-fj7m3      20m (1%)      200m (10%)  20Mi (0%)        100Mi (1%)
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  CPU Requests    CPU Limits    Memory Requests    Memory Limits
  ------------    ----------    ---------------    -------------
  680m (34%)      400m (20%)    920Mi (12%)        1070Mi (14%)
```

从上面的输出可以看出，如果一个pod需要超过1120mCPU和6.23Gi的内存，pod将不会调度到该节点。从`Pods`中可以看出该节点上其他pod使用的资源情况。节点可以为pod提供的资源总和低于节点容量，因为系统守护进程占用了部分资源。`allocatable`给出了节点可以为pod提供的可用资源。可以配置资源配额以限制所有资源的消耗，结和命名空间，可以避免一个团队消耗过多的资源。

#### 容器终止

你的容器可能因为资源匮乏而终止。使用`kubectl describe pod`命令可以查看一个容器是否因为使用超过了限制的资源而被杀死：

```shell
kubectl describe pod simmemleak-hra99
Name:                           simmemleak-hra99
Namespace:                      default
Image(s):                       saadali/simmemleak
Node:                           kubernetes-node-tf0f/10.240.216.66
Labels:                         name=simmemleak
Status:                         Running
Reason:
Message:
IP:                             10.244.2.75
Replication Controllers:        simmemleak (1/1 replicas created)
Containers:
  simmemleak:
    Image:  saadali/simmemleak
    Limits:
      cpu:                      100m
      memory:                   50Mi
    State:                      Running
      Started:                  Tue, 07 Jul 2015 12:54:41 -0700
    Last Termination State:     Terminated
      Exit Code:                1
      Started:                  Fri, 07 Jul 2015 12:54:30 -0700
      Finished:                 Fri, 07 Jul 2015 12:54:33 -0700
    Ready:                      False
    Restart Count:              5
Conditions:
  Type      Status
  Ready     False
Events:
  FirstSeen                         LastSeen                         Count  From                              SubobjectPath                       Reason      Message
  Tue, 07 Jul 2015 12:53:51 -0700   Tue, 07 Jul 2015 12:53:51 -0700  1      {scheduler }                                                          scheduled   Successfully assigned simmemleak-hra99 to kubernetes-node-tf0f
  Tue, 07 Jul 2015 12:53:51 -0700   Tue, 07 Jul 2015 12:53:51 -0700  1      {kubelet kubernetes-node-tf0f}    implicitly required container POD   pulled      Pod container image "k8s.gcr.io/pause:0.8.0" already present on machine
  Tue, 07 Jul 2015 12:53:51 -0700   Tue, 07 Jul 2015 12:53:51 -0700  1      {kubelet kubernetes-node-tf0f}    implicitly required container POD   created     Created with docker id 6a41280f516d
  Tue, 07 Jul 2015 12:53:51 -0700   Tue, 07 Jul 2015 12:53:51 -0700  1      {kubelet kubernetes-node-tf0f}    implicitly required container POD   started     Started with docker id 6a41280f516d
  Tue, 07 Jul 2015 12:53:51 -0700   Tue, 07 Jul 2015 12:53:51 -0700  1      {kubelet kubernetes-node-tf0f}    spec.containers{simmemleak}         created     Created with docker id 87348f12526a
```

上面例子中的 `Restart Count: 5` 说明 `simmemleak`容器已经终止并被重启5次。可以使用`kubectl get pod`命令配合 `-o go-template=...`参数来获取之前处于停止状态的容器。

```shell
kubectl get pod -o go-template='{{range.status.containerStatuses}}{{"Container Name: "}}{{.name}}{{"\r\nLastState: "}}{{.lastState}}{{end}}'  simmemleak-hra99
Container Name: simmemleak
LastState: map[terminated:map[exitCode:137 reason:OOM Killed startedAt:2015-07-07T20:58:43Z finishedAt:2015-07-07T20:58:43Z containerID:docker://0e4095bba1feccdfe7ef9fb6ebffe972b4b14285d5acdec6f0d3ae8a22fad8b2]]
```

容器终止的原因是 `reason:OOM Killed`,  `OOM` 是标准的内存溢出错误，说明容器使用了超过限制的内存。

### 本地短暂存储

**特性状态:** `Kubernetes v1.13` [beta](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/#)

Kubernetes在1.8版本引入了一种新的资源：*ephemeral-storage*用于管理本地短暂存储。在Kubernetes的每一个节点，kubelet的根目录（默认路径为/var/lib/kubelet）和日志目录（/var/log）默认存储在节点的根分区之下。Pod通过emptyDis卷。容器日志，镜像层和镜像可写层共享和消耗节点更分区空间。

该分区是“短暂”的，应用程序不能期望从该分区得到任何性能SLAs，比如磁盘IOPS。本地短暂存储管理仅仅适用于根分区，镜像层和可写层的可选分区不是管理范围内。

> 注意：如果使用了可选的运行时分区，根分区将不会保存任何镜像层和镜像可写层。

#### 本地短暂存储request和limits的设置

Pod中的容器支持以下一个或多个配置：

- `spec.containers[].resources.limits.ephemeral-storage`
- `spec.containers[].resources.requests.ephemeral-storage`

 `ephemeral-storage` 的单位为字节，可以使用普通整数和科学记数法配合后缀 E, P, T, G, M, K或 Ei, Pi, Ti, Gi, Mi, Ki来表示limits和requests。以下多种表达方式给出的值是一样的：

```shell
128974848, 129e6, 129M, 123Mi
```

例如，下面的pod有两个容器，每个容器需要本地短暂存储的大小为2GiB，每个容器限制使用的本地短暂存储的大小为4GiB。对于Pod来说，Pod需要最少4GiB本地短暂存储，最多8GiB本地短暂存储。

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
        ephemeral-storage: "2Gi"
      limits:
        ephemeral-storage: "4Gi"
  - name: wp
    image: wordpress
    resources:
      requests:
        ephemeral-storage: "2Gi"
      limits:
        ephemeral-storage: "4Gi"
```

#### 如果调度对本地短暂存储有要求的Pod

当你创建一个pod时，kubernete的调度器会选择一个节点运行pod。每个节点都有可供Pod使用的本地短暂存储的总量。调度器保证待调度的容器对本地短暂存储的需求的总和不会超过节点的容量。

#### 如何允许对本地短暂存储有限制的Pod

对于容器级别的隔离，如果一个容器可写层和日志使用了超过限制的本地短暂存储，Pod将会被从当前节点驱逐。对于Pod级别的隔离，如果所有容器使用的本地短暂存储总和以及Pod的emptyDis卷超过限制，Pod将会被驱逐。

### 扩展资源

扩展资源是kubernetes.io域之外所有资源的完全限定名。它们允许集群运营商进行增强，并允许用户使用非Kubernetes内置资源。使用扩展资源需要两个步骤：1、集群运营商提供一种扩展资源；2、用户在Pod内请求使用扩展资源。

#### 管理扩展资源

##### 节点级别的扩展资源

节点级别扩展资源与节点相关联。

##### 设备插件管理资源

参考[Device Plugin](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/) 

##### 其他资源

要增减新的节点级扩展资源，群集操作员可以向API服务器提交`PATCH` HTTP请求，以指定群集中节点的`status.capacity`中的可用数量。 执行此操作后，节点的`status.capacity`将包含新资源。 kubelet将异步地使用新资源自动更新`status.allocatable`字段。 请注意，由于调度程序在评估Pod适应性时使用节点status.allocatable值，因此在使用新资源修补节点容量与请求在该节点上调度资源的第一个Pod之间可能存在短暂的延迟。

**样例:**

下面是一个使用curl向集群`k8s-master`中的 `k8s-node-1` 节点添加一个初始容量为5的名为“example.com/foo”的扩展资源。

```shell
curl --header "Content-Type: application/json-patch+json" \
--request PATCH \
--data '[{"op": "add", "path": "/status/capacity/example.com~1foo", "value": "5"}]' \
http://k8s-master:8080/api/v1/nodes/k8s-node-1/status
```

#### 集群级别的扩展资源

集群级别的扩展资源不与节点关联，通常由调度器的扩展程序负责管理，调度器扩展程序同时负责资源消耗和资源配额。

使用 [scheduler policy configuration](https://github.com/kubernetes/kubernetes/blob/release-1.10/pkg/scheduler/api/v1/types.go#L31) 可以配置需要被调度器扩展程序管理的扩展资源。

**样例:**

下面的调度器策略配置表明集群级别的扩展资源“example.com/foo”将有调度器扩展程序负责管理。

- 当一个pod请求 “example.com/foo”时才会使用调度器扩展程序调度。
- `ignoredByScheduler` 字段指明调度器在  `PodFitsResources` 判断时忽略“example.com/foo” 资源检查。

```json
{
  "kind": "Policy",
  "apiVersion": "v1",
  "extenders": [
    {
      "urlPrefix":"<extender-endpoint>",
      "bindVerb": "bind",
      "managedResources": [
        {
          "name": "example.com/foo",
          "ignoredByScheduler": true
        }
      ]
    }
  ]
}
```

#### 消耗扩展资源

用户可以像CPU和内存一样在Pod中消耗扩展资源。调度器负责对资源进行统计，以避免将可用的资源同时分配给多个Pod。

API服务器将扩展资源的数量限制为整数，`3`，`3000m`和`3Ki`都是合法的表达式，`0.5`和`1500`是不合法的表达式。

> **注意** 扩展资源取代了Opaque Integer Resources。 用户可以使用保留的kubernetes.io以外的任何域名前缀。

在Pod内消耗扩展资源，只需要将资源名作为`spec.containers[].resources.limits`的键，

> **注意:** 扩展资源不能过量使用，因此如果容器中配置request和limit，则两者的值必须相等。

一个Pod只有当包括CPU，内存和任何扩展资源都满足的条件下才能被调度。如果Pod要求的资源得不到要求，则Pod的状态将一直保持在`PENDING`。

**样例:**

下面的Pod要求2CPU计算资源和1 “example.com/foo“扩展资源。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
  - name: my-container
    image: myimage
    resources:
      requests:
        cpu: 2
        example.com/foo: 1
      limits:
        example.com/foo: 1
```

### 改进计划

------

Kubernetes 1.5版本仅允许在容器上指定资源数量。计划改进对Pod中所有容器共享的资源的“计费”，例如emptyDir卷。

Kubernetes 1.5版本仅仅支持容器对CPU和内存的request和limit，计划增加新的资源类型，包括磁盘空间，并添加一个支持用户自定义资源的框架。

Kubernetes通过支持多个级别的服务质量来支持资源的过度使用。

Kubernetes 1.5版本中一单位CPU在不同的云供应商和同一云供应商不同类型机器上的含义不相同。例如，在AWS上，节点的容量以ECUs表示，在GCE上以逻辑核表示。我们计划修改cpu资源的定义，以便在供应商和平台之间实现更高的一致性。

