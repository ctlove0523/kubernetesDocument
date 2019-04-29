# 中断

本指南适用于想构建高可用应用程序的应用所有者，因此需要了解Pod可能发生的中断。这也适用于希望集群执行自动操作的集群管理员，例如升级和自动缩放群集。

## 自愿和非自愿中断

Pod不会消失，除非有人（人或控制器）摧毁他们，或者发生了不可避免的硬件或系统错误。

我们将这些不可避免的场景称为对应用程序的*非自愿中断* 。例如：

- 支持节点的物理机发生硬件故障。
- 系统管理员错误的删除了虚拟机。
- 云供应商或超级线程错误导致虚拟机小时。
- 内核恐慌。
- 由于集群网络分区导致节点从集群消失。
- 由于节点资源不足而导致pod被驱逐。

除资源不足外，大多数用户都应熟悉所有这些条件; 它们不是Kubernetes特有的。我们把其他场景称为*自愿中断* 。自愿中断包括应用程序所有者启动的操作和集群管理员启动的操作。应用所有者典型的操作包括：

- 删除deployment或其他管理pod的控制器。
- 更新pod的deployment导致重启。
- 直接删除pod。

集群管理员的操作包括：

- 排空节点进行修复或升级。
- 从集群中排除节点来对集群缩容。
- 从节点移除Pod来保证其他Pod可以部署在该节点。

这些动作可能是集群管理员直接操作，也可能是由管理员允许的自动化执行，或者集群的托管供应商执行。咨询您的集群管理员或咨询您的云提供商或分发文档，以确定是否为您的集群启用了任何自愿中断源。如果没有启用任何自愿中断，你可以跳过Pod中断的预算。

> **警告**：不是所有的自愿中断都受 Pod Disruption Budgets约束。例如，删除deployment或pod会绕过Pod Disruption Budgets。

## 处理中断

以下是一些缓解非自愿中断的方法：

- 确保Pod需要的资源。
- 如果你需要更高的可靠性，对应用进行备份。
- 为了在运行复制的应用程序时获得更高的可用性，应用程序可以跨机架（使用亲和反亲和性）或跨区（跨区集群）部署。

自愿中断的频率各不相同。在基本的Kubernetes集群上，根本不存在自愿中断。但是，集群管理员和集群托管商可能运行额外的服务而导致自愿中断。例如：节点软件滚动升级可能导致自愿中断。此外，集群自动扩容的实现可能导致自愿中断。 您的集群管理员或托管服务提供商应记录预期的自愿中断级别（如果有的话）。Kubernetes提供的功能可以帮助您在频繁的自愿中断的同时运行高可用性应用程序。 我们将这组功能称为*中断预算*。

## PDB如何工作

------

应用程序所有者可以为每个应用程序创建一个`PodDisruptionBudget`对象（PDB）。PDB限制由于自愿中断导致的应用程序同时下线的Pod数量。例如，基于仲裁的应用程序希望确保运行的副本数量永远不会低于仲裁所需的数量。Web前端可能希望确保服务负载的副本数量永远不会低于总数的某个百分比。

集群管理器和集群供应商应该使用遵守Pod Disruption Budget的工具，工具通过调用`Eviction API`  而不是直接删除pod或deployment。例如，使用 `kubectl drain` 命令行工具，在 Kubernetes-on-GCE 集群上可以使用升级脚本`(cluster/gce/upgrade.sh)`

当集群管理员想排除一个节点时，管理员应该使用`kubectl drain` 命令，工具会尝试驱逐机器上的所有pod。Pod驱逐请求可能被暂时拒绝，工具会周期性的尝试失败的请求直到所有的pod被停止或达到配置的超时间。

PDB指定的是应用程序可以容忍的副本数量，相对于预期的副本数量。例如，deployment中的 `.spec.replicas: 5` 字段希望在任何时刻都有5个Pod，如果它的PDB允许在某个时刻有4个Pod，那么Eviction API在某一时刻允许一个pod自愿中断，但是不允许两个pod自愿中断。

构成应用的一组pod由标签选择器选择，这和应用的控制器（deployment、stateful-set等）选择pod是一样的。

期望pod的数量是由pod控制器的 `.spec.replicas`  字段计算。Pod使用对象的`.metadata.ownerReferences` 发现控制器。

PDB不能防止非自愿中断发生，但它们确实违背了预算。

由于应用程序滚动升级导致pod被删除或不可用，确实会被记录到中断预算中，但是在滚动升级时控制器不受PDB的限制，在滚动升级过程中错误的处理是在控制器的spec中定义。

当使用驱逐API来驱逐一个pod时，pod可以被优雅的停止。

## PDB Example

Consider a cluster with 3 nodes, `node-1` through `node-3`. The cluster is running several applications. One of them has 3 replicas initially called `pod-a`, `pod-b`, and `pod-c`. Another, unrelated pod without a PDB, called `pod-x`, is also shown. Initially, the pods are laid out as follows:

| node-1            | node-2            | node-3            |
| ----------------- | ----------------- | ----------------- |
| pod-a *available* | pod-b *available* | pod-c *available* |
| pod-x *available* |                   |                   |

All 3 pods are part of a deployment, and they collectively have a PDB which requires there be at least 2 of the 3 pods to be available at all times.

For example, assume the cluster administrator wants to reboot into a new kernel version to fix a bug in the kernel. The cluster administrator first tries to drain `node-1` using the `kubectl drain` command. That tool tries to evict `pod-a` and `pod-x`. This succeeds immediately. Both pods go into the `terminating` state at the same time. This puts the cluster in this state:

| node-1 *draining*   | node-2            | node-3            |
| ------------------- | ----------------- | ----------------- |
| pod-a *terminating* | pod-b *available* | pod-c *available* |
| pod-x *terminating* |                   |                   |

The deployment notices that one of the pods is terminating, so it creates a replacement called `pod-d`. Since `node-1` is cordoned, it lands on another node. Something has also created `pod-y` as a replacement for `pod-x`.

(Note: for a StatefulSet, `pod-a`, which would be called something like `pod-1`, would need to terminate completely before its replacement, which is also called `pod-1` but has a different UID, could be created. Otherwise, the example applies to a StatefulSet as well.)

Now the cluster is in this state:

| node-1 *draining*   | node-2            | node-3            |
| ------------------- | ----------------- | ----------------- |
| pod-a *terminating* | pod-b *available* | pod-c *available* |
| pod-x *terminating* | pod-d *starting*  | pod-y             |

At some point, the pods terminate, and the cluster looks like this:

| node-1 *drained* | node-2            | node-3            |
| ---------------- | ----------------- | ----------------- |
|                  | pod-b *available* | pod-c *available* |
|                  | pod-d *starting*  | pod-y             |

At this point, if an impatient cluster administrator tries to drain `node-2` or `node-3`, the drain command will block, because there are only 2 available pods for the deployment, and its PDB requires at least 2. After some time passes, `pod-d` becomes available.

The cluster state now looks like this:

| node-1 *drained* | node-2            | node-3            |
| ---------------- | ----------------- | ----------------- |
|                  | pod-b *available* | pod-c *available* |
|                  | pod-d *available* | pod-y             |

Now, the cluster administrator tries to drain `node-2`. The drain command will try to evict the two pods in some order, say `pod-b`first and then `pod-d`. It will succeed at evicting `pod-b`. But, when it tries to evict `pod-d`, it will be refused because that would leave only one pod available for the deployment.

The deployment creates a replacement for `pod-b` called `pod-e`. Because there are not enough resources in the cluster to schedule `pod-e` the drain will again block. The cluster may end up in this state:

| node-1 *drained* | node-2            | node-3            | *no node*       |
| ---------------- | ----------------- | ----------------- | --------------- |
|                  | pod-b *available* | pod-c *available* | pod-e *pending* |
|                  | pod-d *available* | pod-y             |                 |

At this point, the cluster administrator needs to add a node back to the cluster to proceed with the upgrade.

You can see how Kubernetes varies the rate at which disruptions can happen, according to:

- how many replicas an application needs
- how long it takes to gracefully shutdown an instance
- how long it takes a new instance to start up
- the type of controller
- the cluster’s resource capacity

## Separating Cluster Owner and Application Owner Roles

Often, it is useful to think of the Cluster Manager and Application Owner as separate roles with limited knowledge of each other. This separation of responsibilities may make sense in these scenarios:

- when there are many application teams sharing a Kubernetes cluster, and there is natural specialization of roles
- when third-party tools or services are used to automate cluster management

Pod Disruption Budgets support this separation of roles by providing an interface between the roles.

If you do not have such a separation of responsibilities in your organization, you may not need to use Pod Disruption Budgets.

## How to perform Disruptive Actions on your Cluster

If you are a Cluster Administrator, and you need to perform a disruptive action on all the nodes in your cluster, such as a node or system software upgrade, here are some options:

- Accept downtime during the upgrade.
- Failover to another complete replica cluster.
  - No downtime, but may be costly both for the duplicated nodes, and for human effort to orchestrate the switchover.
- Write disruption tolerant applications and use PDBs.
  - No downtime.
  - Minimal resource duplication.
  - Allows more automation of cluster administration.
  - Writing disruption-tolerant applications is tricky, but the work to tolerate voluntary disruptions largely overlaps with work to support autoscaling and tolerating involuntary disruptions.

## What's next

- Follow steps to protect your application by [configuring a Pod Disruption Budget](https://kubernetes.io/docs/tasks/run-application/configure-pdb/).
- Learn more about [draining nodes](https://kubernetes.io/docs/tasks/administer-cluster/safely-drain-node/)

## Feedback
