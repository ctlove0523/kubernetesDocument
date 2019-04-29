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

## Dealing with Disruptions

Here are some ways to mitigate involuntary disruptions:

- Ensure your pod [requests the resources](https://kubernetes.io/docs/tasks/configure-pod-container/assign-cpu-ram-container) it needs.
- Replicate your application if you need higher availability. (Learn about running replicated [stateless](https://kubernetes.io/docs/tasks/run-application/run-stateless-application-deployment/) and [stateful](https://kubernetes.io/docs/tasks/run-application/run-replicated-stateful-application/) applications.)
- For even higher availability when running replicated applications, spread applications across racks (using [anti-affinity](https://kubernetes.io/docs/user-guide/node-selection/#inter-pod-affinity-and-anti-affinity-beta-feature)) or across zones (if using a [multi-zone cluster](https://kubernetes.io/docs/setup/multiple-zones).)

The frequency of voluntary disruptions varies. On a basic Kubernetes cluster, there are no voluntary disruptions at all. However, your cluster administrator or hosting provider may run some additional services which cause voluntary disruptions. For example, rolling out node software updates can cause voluntary disruptions. Also, some implementations of cluster (node) autoscaling may cause voluntary disruptions to defragment and compact nodes. Your cluster administrator or hosting provider should have documented what level of voluntary disruptions, if any, to expect.

Kubernetes offers features to help run highly available applications at the same time as frequent voluntary disruptions. We call this set of features *Disruption Budgets*.

## How Disruption Budgets Work

An Application Owner can create a `PodDisruptionBudget` object (PDB) for each application. A PDB limits the number of pods of a replicated application that are down simultaneously from voluntary disruptions. For example, a quorum-based application would like to ensure that the number of replicas running is never brought below the number needed for a quorum. A web front end might want to ensure that the number of replicas serving load never falls below a certain percentage of the total.

Cluster managers and hosting providers should use tools which respect Pod Disruption Budgets by calling the [Eviction API](https://kubernetes.io/docs/tasks/administer-cluster/safely-drain-node/#the-eviction-api)instead of directly deleting pods or deployments. Examples are the `kubectl drain` command and the Kubernetes-on-GCE cluster upgrade script (`cluster/gce/upgrade.sh`).

When a cluster administrator wants to drain a node they use the `kubectl drain` command. That tool tries to evict all the pods on the machine. The eviction request may be temporarily rejected, and the tool periodically retries all failed requests until all pods are terminated, or until a configurable timeout is reached.

A PDB specifies the number of replicas that an application can tolerate having, relative to how many it is intended to have. For example, a Deployment which has a `.spec.replicas: 5` is supposed to have 5 pods at any given time. If its PDB allows for there to be 4 at a time, then the Eviction API will allow voluntary disruption of one, but not two pods, at a time.

The group of pods that comprise the application is specified using a label selector, the same as the one used by the application’s controller (deployment, stateful-set, etc).

The “intended” number of pods is computed from the `.spec.replicas` of the pods controller. The controller is discovered from the pods using the `.metadata.ownerReferences` of the object.

PDBs cannot prevent [involuntary disruptions](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/#voluntary-and-involuntary-disruptions) from occurring, but they do count against the budget.

Pods which are deleted or unavailable due to a rolling upgrade to an application do count against the disruption budget, but controllers (like deployment and stateful-set) are not limited by PDBs when doing rolling upgrades – the handling of failures during application updates is configured in the controller spec. (Learn about [updating a deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#updating-a-deployment).)

When a pod is evicted using the eviction API, it is gracefully terminated (see `terminationGracePeriodSeconds` in [PodSpec](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.14/#podspec-v1-core).)

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