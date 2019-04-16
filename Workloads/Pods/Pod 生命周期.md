# Pod 生命周期



## Pod 阶段

A Pod’s `status` field is a [PodStatus](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.14/#podstatus-v1-core) object, which has a `phase` field.

Pod的`status` 字段是一个`PodStats` 对象，包含有一个`phase` 字段。

The phase of a Pod is a simple, high-level summary of where the Pod is in its lifecycle. The phase is not intended to be a comprehensive roll up of observations of Container or Pod state, nor is it intended to be a comprehensive state machine.

Pod的阶段是简单的，高度概括了pod处于生命周期的哪个阶段。该阶段并非旨在全面汇总Container或Pod状态的观察结果，也不打算成为一个综合状态机。

The number and meanings of Pod phase values are tightly guarded. Other than what is documented here, nothing should be assumed about Pods that have a given `phase` value.

Pod阶段的数量和含义受到严密的保护，除了这里记录的内容之外，没有任何关于具有给定的Pod阶段的假设。

下面是`phase` 的可能取值:

| Value              | Description                                                  |
| ------------------ | ------------------------------------------------------------ |
| `Pending`          | Kubernetes系统已经接收该pod，但尚未创建一个或多个容器镜像。这包括调度之前的时间以及通过网络下载镜像的时间，pod可能一段时间内处于该阶段。 |
| `Running`          | Pod已经绑定到节点，所有的容器已经被闯将，至少有一个容器在允许或在启动的过程中又或者pod在重启。 |
| `Succeeded`        | Pod内的所有容已成功终止并且不会重启。                        |
| `Failed`           | Pod内的所有容器都已经停止，并且至少一个容器以失败而停止。这可能是容器以非0状态退出或者容器被系统停止。 |
| `Unknown`          | 因为某种原因无法获取pod的转台，通常是由于和pod的主机通信错误导致。 |
| `Completed`        | Pod已经运行完成，没有任何事情需要pod运行。                   |
| `CrashLoopBackOff` | Pod内至少有一个容器以非成功状态退出。                        |

## Pod 条件

A Pod has a PodStatus, which has an array of [PodConditions](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.14/#podcondition-v1-core) through which the Pod has or has not passed. Each element of the PodCondition array has six possible fields:

每个Pod都有一个PodStatus，PodStatus有一个PodConditions 数组，记录了Pod通过或没通过的条件。每一个PodCondition数组中的元素有6个可能的字段：

| 字段                 | 含义                                             |
| -------------------- | ------------------------------------------------ |
| `lastProbeTime`      | 上次探测Pod条件的时间戳                          |
| `lastTransitionTime` | 上次Pod状态变换的时间戳                          |
| `message`            | 用户可读的有关状态变换的详细信息                 |
| `reason`             | 唯一，一个单词，驼峰的描述最后一次状态变换的原因 |
| `status`             | 可能取值`True` `False` 和`Unknown`               |
| `type`               | 可能的取值PodScheduled-                          |



-  `lastProbeTime` 上次探测Pod条件的时间戳。
-  `lastTransitionTime` 上次Pod状态变换的时间戳。
-  `message` 用户可读的有关状态变换的信息。
-  `reason` 唯一，单个单词，驼峰的描述最后一次状态变换的原因。
-  `status` 可能取值“`True`”, “`False`”, 和 “`Unknown`”.
-  `type` 可能的取值如下:
  - `PodScheduled`:   Pod已经被调度到了一个节点;
  - `Ready`:  Pod能够提供请求，应该添加到所有匹配服务的负载平衡池中;
  - `Initialized`: 所有init容器都已成功启动;
  - `Unschedulable`:  调度程序现在无法调度Pod，例如由于缺乏资源或其他约束;
  - `ContainersReady`: Pod内所有的容器都已经Ok.

## 容器探针

A [Probe](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.14/#probe-v1-core) is a diagnostic performed periodically by the [kubelet](https://kubernetes.io/docs/admin/kubelet/) on a Container. To perform a diagnostic, the kubelet calls a [Handler](https://godoc.org/k8s.io/kubernetes/pkg/api/v1#Handler)implemented by the Container. There are three types of handlers:

- [ExecAction](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.14/#execaction-v1-core): Executes a specified command inside the Container. The diagnostic is considered successful if the command exits with a status code of 0.
- [TCPSocketAction](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.14/#tcpsocketaction-v1-core): Performs a TCP check against the Container’s IP address on a specified port. The diagnostic is considered successful if the port is open.
- [HTTPGetAction](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.14/#httpgetaction-v1-core): Performs an HTTP Get request against the Container’s IP address on a specified port and path. The diagnostic is considered successful if the response has a status code greater than or equal to 200 and less than 400.

Each probe has one of three results:

- Success: The Container passed the diagnostic.
- Failure: The Container failed the diagnostic.
- Unknown: The diagnostic failed, so no action should be taken.

The kubelet can optionally perform and react to two kinds of probes on running Containers:

- `livenessProbe`: Indicates whether the Container is running. If the liveness probe fails, the kubelet kills the Container, and the Container is subjected to its [restart policy](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#restart-policy). If a Container does not provide a liveness probe, the default state is `Success`.
- `readinessProbe`: Indicates whether the Container is ready to service requests. If the readiness probe fails, the endpoints controller removes the Pod’s IP address from the endpoints of all Services that match the Pod. The default state of readiness before the initial delay is `Failure`. If a Container does not provide a readiness probe, the default state is `Success`.

### When should you use liveness or readiness probes?

If the process in your Container is able to crash on its own whenever it encounters an issue or becomes unhealthy, you do not necessarily need a liveness probe; the kubelet will automatically perform the correct action in accordance with the Pod’s `restartPolicy`.

If you’d like your Container to be killed and restarted if a probe fails, then specify a liveness probe, and specify a `restartPolicy`of Always or OnFailure.

If you’d like to start sending traffic to a Pod only when a probe succeeds, specify a readiness probe. In this case, the readiness probe might be the same as the liveness probe, but the existence of the readiness probe in the spec means that the Pod will start without receiving any traffic and only start receiving traffic after the probe starts succeeding. If your Container needs to work on loading large data, configuration files, or migrations during startup, specify a readiness probe.

If you want your Container to be able to take itself down for maintenance, you can specify a readiness probe that checks an endpoint specific to readiness that is different from the liveness probe.

Note that if you just want to be able to drain requests when the Pod is deleted, you do not necessarily need a readiness probe; on deletion, the Pod automatically puts itself into an unready state regardless of whether the readiness probe exists. The Pod remains in the unready state while it waits for the Containers in the Pod to stop.

For more information about how to set up a liveness or readiness probe, see [Configure Liveness and Readiness Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/).

## Pod and Container status

For detailed information about Pod Container status, see [PodStatus](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.14/#podstatus-v1-core) and [ContainerStatus](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.14/#containerstatus-v1-core). Note that the information reported as Pod status depends on the current [ContainerState](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.14/#containerstatus-v1-core).

## Container States