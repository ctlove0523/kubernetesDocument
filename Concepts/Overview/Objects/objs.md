# 理解 Kubernetes 对象

在本文中解释对象在K8s的API中的表示以及如何使用`.yaml` 文件描述。

## 理解 Kubernetes 对象

*K8s 对象*是K8s系统中的持久化实体，K8s 使用这些实体表示集群状态。特别的，对象可以表示以下状态：

- 有哪些容器应用在运行，运行在哪些节点上
- 应用可用资源
- 应用行为策略，重启策略，升级和容错策略等。

一个K8s对象就是一个*期望记录* ，一旦创建了该对象，K8s系统会不间断的工作来确保对象存在。通过创建对象，你有效的告诉了K8s 系统你所期望的集群工作负载，这也是集群的合适（最终）状态。

处理Kubernets对象（不管是通过创建、修改或者删除对象），你都需要使用K8s的API。当你使用`kubectl` 命令行接口时，CLI符合调用合适的K8s API。你也可以通过客户端库直接在应用中使用K8s API。

### 对象描述和状态

每个K8s对象都包含两个嵌套对象字段，用于控制对象的配置：对象描述和对象状态。 您必须提供的对象描述描述了对象的期望状态。 状态描述对象的实际状态，由K8s系统提供和更新。 在任何给定时间，K8s控制平面都会主动管理对象的实际状态，以匹配您提供的期望状态。

例如，K8s Deployment 是一个可以描述集群上运行的应用的对象。当创建Deployment对象时，可能指定期望应用有三个副本在运行。Kubertest读取Deployment的信息并启动三个应用实例，如果某个实例因为故障停止允许，K8s系统可以识别到实际副本和期望数据的差别，并启动一个新的应用实例。

### 描述 K8s 对象

当你创建一个K8s对象时，你必须提供`spec` 来描述对象期望的状态，当然也包括一些对象的基本信息（比如，对象的名字）。当使用K8s API创建对象（或者通过kubetl简介创建），API要求对象所有的信息必须以JSON的格式通过请求体传输。**更常见的时为kubectl提供一个.yaml文件**，kubectl 将yaml中的信息转换为JSON格式。

下面是一个创建K8s Deployment对象的实例:

```yaml
apiVersion: apps/v1 # for versions before 1.9.0 use apps/v1beta2
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2 # tells deployment to run 2 pods matching the template
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```

使用`Kubectl`创建Deployment的样例如下:

```shell
kubectl create -f deployment.yaml --record
```

命令的输出和下面类似:

```shell
deployment.apps/nginx-deployment created
```

### 必选字段

在对象的`。yaml` 文件中，以下字段为必须字段：

- `apiVersion` - 创建对象使用的API版本
- `kind` - 对象类型
- `metadata` - 对象元数据，这些信息有助于识别对象，包括 `name` 和可选的`namespace`  对象。

在`.yaml` 文件中也要包含`spec`字段，`spec`字段的格式因对象不同而不同，同时包含与对象相关的内嵌字段。不同对象的`spec` 格式可以参考 [Kubernetes API Reference](https://K8s.io/docs/reference/generated/K8s-api/v1.13/) 。