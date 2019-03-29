# StatefulSet Basiscs

## 目标

StatefulSets旨在与有状态应用程序和分布式系统一起使用。但是，在Kubernetes上管理有状态应用程序和分布式系统是一个广泛而复杂的主题。为了演示StatefulSet的基本功能，而不是将前一个主题与后者混淆，您将使用StatefulSet部署一个简单的Web应用程序。

学习完本教程后，你将掌握以下内容.

- 如何创建 StatefulSet
- 了解StatefulSet如何管理自己的Pods
- 如何删除 StatefulSet
- 如何扩展 StatefulSet
- 如何升级 StatefulSet 的Pods

## 准备工作

在开始学习本篇教程前，你需要熟悉Kubernetes的以下概念：

- Pods
- Cluster DNS
- Headless Services
- PersistentVolumes
- PersistentVolume Provisioning
- StatefulSets
- kubectl CLI

本教程假定您的群集已配置为动态设置PersistentVolumes。 如果您的群集未配置为执行此操作，则必须在开始本教程之前手动配置两个1 GiB卷。

创建 StatefulSet

使用下面的web.yaml 创建一个web应用。

    apiVersion: v1
    kind: Service
    metadata:
      name: nginx
      labels:
        app: nginx
    spec:
      ports:
      - port: 80
        name: web
      clusterIP: None
      selector:
        app: nginx
    ---
    apiVersion: apps/v1
    kind: StatefulSet
    metadata:
      name: web
    spec:
      serviceName: "nginx"
      replicas: 2
      selector:
        matchLabels:
          app: nginx
      template:
        metadata:
          labels:
            app: nginx
        spec:
          containers:
          - name: nginx
            image: k8s.gcr.io/nginx-slim:0.8
            ports:
            - containerPort: 80
              name: web
            volumeMounts:
            - name: www
              mountPath: /usr/share/nginx/html
      volumeClaimTemplates:
      - metadata:
          name: www
        spec:
          accessModes: [ "ReadWriteOnce" ]
          resources:
            requests:
              storage: 1Gi

通过下面的命令创建定义在web.yaml 中的Headless Service 和 StatefulSet ：

    kubectl apply -f web.yaml

### 定义Pod创建顺序

对于一个拥有N个副本的StatefulSet，当部署Pods时，Pod按照（0，N-1）的线性顺序创建。第n个pod只有在第n-1个pod处于Running和Ready状态时才能被启动。



StatefulSet 中的Pod

StatefulSet中的Pod具有唯一的序数索引和稳定的网络标识。


