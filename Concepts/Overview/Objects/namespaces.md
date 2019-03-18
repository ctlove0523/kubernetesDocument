# 命名空间

在同一个物理集群内，K8s支持多个虚拟集群，这些虚拟集群被称为命名空间。

- 何时使用多命名空间
- 使用命名空间
- 命名空间 和 DNS
- 不是所有对象都在命名空间内

## 何时使用多命名空间

命名空间旨在用于多个用户分布在多个团队或项目中的环境中。只有十几个用户的集群，没有创建或考虑使用命名空间的必要。当你需要使用命名空间提供的功能时再使用命名空间。命名空间为名字提供了范围，资源的名字在一个命名空间内需要唯一，跨多个命名空间不要求唯一。

命名空间是一种在多个用户之间划分集群资源的方式（通过资源配额）。将K8s将来的发行版本中，在同一个命名空间中的对象默认使用一样的接入控制策略。使用命名空间分割略有不同的资源毫无必要，比如同一个软件的多个版本：使用标签区分同一个命名空间的资源更加合适。

## 使用 命名空间

命名空间的创建和删除参考 [Admin Guide documentation for 命名空间](https://kubernetes.io/docs/admin/命名空间).

### 查看 命名空间

使用下面的命令可以查看集群内所有使用的命名空间:

```shell
kubectl get 命名空间
NAME          STATUS    AGE
default       Active    1d
kube-system   Active    1d
kube-public   Active    1d
```

Kubernetes 启动时默认有三个命名空间:

- `default` 默认的命名空间，如果对象没有指定命名空间则都属于default。
- `kube-system` Kubernetes创建的对象在该命名空间。
- `kube-public` 当某些资源需要被集群内的所有用户访问是。该命名空间公开只是仅仅只是一个约定，而不是Kubernetes的要求。

### 为请求设置命名空间

使用--命名空间标志可以设置请求的临时命名空间。

例如:

```shell
kubectl --命名空间=<insert-命名空间-name-here> run nginx --image=nginx
kubectl --命名空间=<insert-命名空间-name-here> get pods
```

### 设置首选 命名空间

您可以在该上下文中为所有后续kubectl命令永久保存命名空间。

```shell
kubectl config set-context $(kubectl config current-context) --命名空间=<insert-命名空间-name-here>
# Validate it
kubectl config view | grep 命名空间:
```

## 命名空间 and DNS

当你创建一个Service时，会自动创建一个与Service关联的DNS实体。DNS实体的格式为：`<service-name>.<命名空间-name>.svc.cluster.local` ，这意味着当容器仅仅使用`<service-name>` 就剋定位到命名空间中的服务。这对于在多个名称空间（如开发，分段和生产）中使用相同的配置非常有用。如果想跨命名空间，必须使用全限定名。

## 不是所有对象都在 命名空间中

大多数的资源都在相同的命名空间中。但是命名空间资源自身不在命名空间中。更低层的资源，比如节点和持久卷不在任何命名空间中。

查看在命名空间和不在命名空间中的资源可以使用下面的命令:

```shell
# In a 命名空间
kubectl api-resources --命名空间d=true

# Not in a 命名空间
kubectl api-resources --命名空间d=false
```

