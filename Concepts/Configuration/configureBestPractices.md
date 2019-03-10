# 配置最佳实践

本文档重点介绍并整合了整个用户指南，入门文档和示例中介绍的配置最佳实践。

- 一般配置提示
- 裸 Pods vs ReplicaSets, Deployments, and Jobs
- Services
- 使用标签
- 容器镜像
- 使用kubectl

## 一般配置提示

- 定义配置时，使用最新稳定版本的API。
- 配置文件在推送到集群之前应当存储在版本控制中。如果需要，这将允许你快速的回退配置修改，这对集群重建和恢复也有帮助。
- 使用YAML而不是JSON编写配置文件。
- 如果合理，将相关的对象定义在一个文件中，一个文件通常要比多个文件更好管理。
- 另外很多`kubectl`可以在目录上执行很多命令
- 如果没有必要不要设置默认值：配置越简单、短小，导致错误的概率就越小。
- 将对象的描述放置到注解中，支持更高的自省。

## 裸体 Pods vs ReplicaSets, Deployments, and Jobs

- 如果可能尽量不要使用与ReplicaSet或Deployment没有关联的裸Pod，如果节点失败，裸Pod不会被重新调度。

一个Deployment会创建一个ReplicaSet来保证期望的Pod副本一直处于可用状态，同时会创建替换Pod的策略，比如滚动升级，这差不多是最完美的直接创建Pod的方法，除了一些显式的`restartPolicy：Never`场景，这种场景使用Job可能更合适。

## Services

- Create a [Service](https://kubernetes.io/docs/concepts/services-networking/service/) before its corresponding backend workloads (Deployments or ReplicaSets), and before any workloads that need to access it. When Kubernetes starts a container, it provides environment variables pointing to all the Services which were running when the container was started. For example, if a Service named `foo` exists, all containers will get the following variables in their initial environment:

```shell
  FOO_SERVICE_HOST=<the host the Service is running on>
  FOO_SERVICE_PORT=<the port the Service is running on>
```

If you are writing code that talks to a Service, don’t use these environment variables; use the [DNS name of the Service](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/) instead. Service environment variables are provided only for older software which can’t be modified to use DNS lookups, and are a much less flexible way of accessing Services.

- Don’t specify a `hostPort` for a Pod unless it is absolutely necessary. When you bind a Pod to a `hostPort`, it limits the number of places the Pod can be scheduled, because each <`hostIP`, `hostPort`, `protocol`> combination must be unique. If you don’t specify the `hostIP` and `protocol` explicitly, Kubernetes will use `0.0.0.0` as the default `hostIP` and `TCP` as the default `protocol`.

If you only need access to the port for debugging purposes, you can use the [apiserver proxy](https://kubernetes.io/docs/tasks/access-application-cluster/access-cluster/#manually-constructing-apiserver-proxy-urls) or [`kubectl port-forward`](https://kubernetes.io/docs/tasks/access-application-cluster/port-forward-access-application-cluster/).

If you explicitly need to expose a Pod’s port on the node, consider using a [NodePort](https://kubernetes.io/docs/concepts/services-networking/service/#nodeport) Service before resorting to `hostPort`.

- Avoid using `hostNetwork`, for the same reasons as `hostPort`.
- Use [headless Services](https://kubernetes.io/docs/concepts/services-networking/service/#headless-services) (which have a `ClusterIP` of `None`) for easy service discovery when you don’t need `kube-proxy` load balancing.

## 使用标签

**标签才是Kubernetes的利器**

- 定义和使用标签作为表示应用和Deployment的语义属性，例如下面的标签 `{ app: myapp, tier: frontend, phase: test, deployment: v3 }` 。可以在其他资源中使用这些标签选择合适的Pod。

通过从选择器中省略特定于发行版的标签，可以使Service跨越多个Deployment。Deployment使得不停机更新服务变得更加容易。

Deployment描述了对象的期望状态，并且如果应用的状态发生了改变，则部署控制器以受控速率将实际状态改变为期望状态。

- 可以修改标签来调试。因为Kubernetes的控制器和Service使用标签选择器选择Pod，从Pod移除了相关的标签会导致控制器或Service的流量不会转发到原来的Pod。如果移除一个已经存在的Pod的标签，控制器会创建一个新的Pod取代原来的Pod。

## 容器镜像

 `imagePullPolicy` 和镜像的标签影响kubelet尝试拉取特定镜像的行为：

- `imagePullPolicy: IfNotPresent`: 本地不存在时拉取.
- `imagePullPolicy: Always`: pod每次启动时都拉取.
- ~~`imagePullPolicy`省略，使用镜像标签要么是latest要么省略应用Always 策略。~~
- ~~`imagePullPolicy`省略，使用镜像标签但不是`latest`应用`IfNotPresent` 策略。~~
- `imagePullPolicy: Never`: 假设本地存在镜像，从不拉取镜像.

> **注意:** 为了确保容器每次都使用相同版本的镜像，可以配置`digest` 。

> **注意:** 在生产环境部署容器时应避免使用`:latest`标签的镜像。

> **注意:** 底层镜像提供者的缓存语义使得`imagePullPolicy: Always` 策略更加高效。举个例子，使用Docker，如果镜像已经在本地存在，拉取请求会非常的迅速因为所有的镜像层都在缓存在本地，无需从镜像中心下载。

## 使用 kubectl

- 使用  `kubectl apply -f <directory>` 或 `kubectl create -f <directory>`。这可以搜索目录下所有的 `.yaml`, `.yml`, 和 `.json` 配置文件然后传递给`apply` 或 `create`。
- 对于`ge`t和`delete` 操作使用标签选择器而不是指定特定对象的名字。
- 使用 `kubectl run` 和 `kubectl expose` 快速创建一个单容器Deployments 和 Services。

