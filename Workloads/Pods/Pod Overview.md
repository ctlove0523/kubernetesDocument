## 理解 Pods

*Pod* 是Kubernetes最基本的构建快，也是用户可以创建、部署的Kubernetes对象模型中最小和最简单的单元。一个*Pod* 代表集群中一个允许的进程。

一个Pod封装了一个应用程序容器（在某些情况下可能是多个容器）、存储资源、一个唯一的网络IP以及控制容器如何允许的选项。Pod表示部署单元：*Kubernetes* 中的单个应用程序实例，可能包含单个容器或少量紧密耦合且共享资源的容器。

> Docker是Kubernetes Pod最常使用的容器运行时环境，但是Pod也支持其他容器运行时。

使用Kubernetes集群中的Pod两种主要方式：

- **Pod 运行单个容器**.  一个Pod一个容器的模式是kubernetes最普遍的使用场景。这这种场景中，你可以理解为一个Pod封装了一个容器，Kubernetes直接管理Pod而不是容器。
- **Pods 允许多个需要一起工作的容器**. Pod可以封装由多个共址容器组成的应用程序，这些容器紧密耦合并需要共享资源。这些共址容器可能形成一个单一的内聚服务单元，一个容器通过共享卷向外提供文件，而另一个“sidecar”容器刷新或更新这些文件。Pod将这些容器和存储资源作为单个可管理实体包装在一起。

The [Kubernetes Blog](http://kubernetes.io/blog) has some additional information on Pod use cases. For more information, see:

- [The Distributed System Toolkit: Patterns for Composite Containers](https://kubernetes.io/blog/2015/06/the-distributed-system-toolkit-patterns)
- [Container Design Patterns](https://kubernetes.io/blog/2016/06/container-design-patterns)

每个Pod都用于运行给定应用程序的单个实例。 如果要水平扩展应用程序（例如，运行多个实例），则应使用多个Pod，每个实例一个。在Kubernetes中，这通常被称为副本。 复制Pod通常由称为Controller的抽象创建和管理。

### Pods 如何管理多个容器

Pod旨在支持多个协作流程（作为容器），形成一个有凝聚力的服务单元。一个Pod内的容器被自动部署和调度到集群内相同的物理机或虚拟机上。容器可以共享存储和依赖，互相通信，并协调容器何时以及如何停止。

Note that grouping multiple co-located and co-managed containers in a single Pod is a relatively advanced use case. You should use this pattern only in specific instances in which your containers are tightly coupled. For example, you might have a container that acts as a web server for files in a shared volume, and a separate “sidecar” container that updates those files from a remote source, as in the following diagram:

请注意，在单个Pod中对多个共址和共同管理的容器进行分组是一个相对高级的使用场景。您应该仅在容器紧密耦合的特定场景中使用此模式。例如，您可能有一个容器充当共享卷中文件的Web服务器，以及一个单独的“sidecar”容器，用于从远程源更新这些文件，如下图所示：

![pod](D:\文档\kubernetes\pod.svg)



#### pod 图

Pods provide two kinds of shared resources for their constituent containers: *networking* and *storage*.

Pod为容器提供两种共享资源：网络和存储。

#### 网络

Each Pod is assigned a unique IP address. Every container in a Pod shares the network namespace, including the IP address and network ports. Containers *inside a Pod* can communicate with one another using `localhost`. When containers in a Pod communicate with entities *outside the Pod*, they must coordinate how they use the shared network resources (such as ports).

每个Pod都会分配一个唯一的IP地址。Pod内的容器共享网络命名空间，包括IP地址和网络端口。Pod内容器之间可以使用`localhost` 进行通信。当Pod内的容器和Pod外部的实体通信时，如果协调容器如何使用共享的网络资源。

#### 存储

A Pod can specify a set of shared storage *volumes*. All containers in the Pod can access the shared volumes, allowing those containers to share data. Volumes also allow persistent data in a Pod to survive in case one of the containers within needs to be restarted. See [Volumes](https://kubernetes.io/docs/concepts/storage/volumes/) for more information on how Kubernetes implements shared storage in a Pod

一个Pod可以指定一组共享卷。Pod内的所有容器都可以访问共享卷，从允许容器间共享数据。如果Pod中的一个容器需要重启，共享卷还支持Pod内持久数据的存储。


在Kubernetes中你很少会创建独立的Pod，即便是单个Pod。这是因为Pod被设计为一种相对短暂的实体。当Pod被创建时（用户直接创建或控制器间接创建），Pod会被调度到集群中的一个节点上。Pod将一直在该节点上直到进程终止、pod对象被删除、由于缺乏资源pod被驱逐，或者节点失败。
