# Kubernetes Components
本文重点介绍构建一个可用Kubernetes集群需要的二进制组件：

- 主要组件
- 节点组件 
- 插件

## 主要组件

主要组件提供Kubernetes集群的控制平面。主要组件负责集群全局决策（比如，调度），检测并响应集群事件（当副本控制器的副本字段的值不满足期望时，启动一个新的Pod）。

主要组件可以运行在集群中的任何机器之上。但是，为了简单起见，通常配置脚本在同一台机器上启动所有主要组件，而且不在该机器上运行用户的容器。

### kube-apiserver

------

暴露Kubernetes API的主要组件。kube-apiserver 是Kubernetes集群的前端控制平面。kube-apiserver可以通过增加实例来实现水平扩展。

### etcd

------

etcd是一种一致性、高可用的键值存储系统，Kubernetes用来存储所有集群数据。你的Kubernetes集群中的etcd应当有备份计划。有关etcd更深入的内容参考etcd documentation.

### kube-scheduler

------

监控为分配节点的新创建Pod，并选择一个合适的节点运行pod。调度需要考虑一下因素：集群资源、硬件/软件/策略约束、亲和/反亲和配置、数据位置、负载间干扰和调度的时间限制。

### kube-controller-manager

------

用于运行控制器的组件。

从逻辑上看，每个控制器都是一个独立的进程，但是为了降低复杂性，它们被编译在一个二进制文件中并运行在一个进程中。

包括以下控制器：

- 节点控制器。检测节点故障并响应。
- 副本控制器。负责为系统中的每个复制控制器对象维护正确数量的pod。
- 端点控制器。填充端点对象（比如连接Service和Pod）
- 服务帐户和令牌控制器。对于新创建的命名空间创建默认账户和API访问令牌。



### cloud-controller-manager

------

[cloud-controller-manager运行和底层云提供商交互的控制器。cloud-controller-manager是Kubernetes在1.6版本引入，目前还是alpha版本功能。

cloud-controller-manager只循环调用云供应商特定的控制器。你需要在kube-controller-manager中禁止这些控制器的循环调用。可以在启动kube-controller-manager时将 `--cloud-provider` 设置为 `external` 关闭循环调用控制器。

cloud-controller-manager 允许云供应商的代码和Kubernetes的代码独立发展，不再互相依赖。在之前的版本中，Kubernetes的核心代码依赖云供应商的特殊代码来提供功能。在将来的版本中，云供应商特有的代码应由云供应商自己维护，在允许Kubernetes时与cloud-controller-manager连接。

以下控制器依赖于云供应商：

* 节点控制器。节点停止响应时，检查节点是否已经从云删除。
* 路由控制器。设置底层云基础设施路由。
* 服务控制器。创建，更新和删除云供应商负载均衡器。
* 卷控制器。创建，附加，挂载卷，并和云供应商交互以协调卷。



## 节点组件

节点组件允许在每一个节点，维护运行的pods并为Kubernetes提供运行时环境。

### kubelet

------

运行在集群中每个节点上的一个代理。kubelet确保pod内的容器运行。

kubelet拥有一系列通过各种机制提供的PodSpecs，并确保这些PodSpecs中描述的容器正在运行且健康。kubelet不会管理非Kubernetes创建的容器。

### kube-proxy

[kube-proxy](https://kubernetes.io/docs/admin/kube-proxy/) enables the Kubernetes service abstraction by maintaining network rules on the host and performing connection forwarding.

kube-proxy通过维护主机上的网络规则并执行连接转发来支持Kubernetes服务抽象。

### Container Runtime

------

容器运行时环境是负责运行容器的软件，Kubernetes支持多种运行时环境：Docker，containerd，cri-o，rktlet和任何 [Kubernetes CRI (Container Runtime Interface)](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-node/container-runtime-interface.md).实现

## 插件

插件是实现了集群某种功能的pod或服务。Pod可能由Depoyment，ReplicationController等管理。命名空间插件在`kube-system` 命名空间内创建。

Kubernetes选定的插件如下，其他支持的插件可以从 [插件列表](https://kubernetes.io/docs/concepts/cluster-administration/addons/).获取。

### DNS

------

虽然其他插件并非一定需要，但所有Kubernetes集群都应具有集群DNS，因为许多示例都依赖于它。集群DNS是一个DNS服务器，除了你环境中的其他DNS服务器，为Kuberntes中的服务提供服务。Kubernetes启动的容器在DNS搜索列表中自动包含该DNS服务器。

### Web UI (Dashboard)

------

仪表板是Kubernetes集群的基于Web的通用UI。 它允许用户管理和调试群集中运行的应用程序以及群集本身。

### Container Resource Monitoring

------

容器资源监控将容器的通用时间序列指标存储在一个中央数据库中，并提供用于浏览该数据的UI。

### Cluster-level Logging

------

集群级别的日志机制支持将所有的容器日志存储到一个中心日志存储中，并提供搜索和查询接口。