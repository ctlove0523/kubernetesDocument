# 标签和选择器

标签是附加到对象（比如 pod）之上的键值对。标签旨在用于指定对用户有意义并且和用户相关对象的标示属性，但是不直接影响核心系统的语义。标签可用于组织和选择对象的子集。标签可以在创建对象的时候附加到对象，并且可以在之后的任何时间添加或修改标签。每个对象都可以定义一组键/值标签，对于给定的一个对象标签的键必须唯一。

    "metadata": {
      "labels": {
        "key1" : "value1",
        "key2" : "value2"
      }
    }

标签支持高效的查询和监控，非常适合在UI和CLI中使用。 非标示信息应使用注释记录。

## 动机

标签使用户能够以松散耦合的方式将用户的组织结构映射到系统对象，而无需客户端存储这些映射关系。

服务的deployment和批量处理管道通常都是多维度实体（比如，多个分区或多个deployment，多层发布路线，多层，每层多个微服务）。管理通常需要交叉操作，这打破了严格的层次表示的封装，特别是由基础设施而不是用户确定的严格的层次结构。

标签样例:

- "release" : "stable", "release" : "canary"
- "environment" : "dev", "environment" : "qa", "environment" : "production"
- "tier" : "frontend", "tier" : "backend", "tier" : "cache"
- "partition" : "customerA", "partition" : "customerB"
- "track" : "daily", "track" : "weekly"

这些只是一些常用的标签，你可以开发自己的框架，但是需要牢记对于一个给定的对象，标签的键必须唯一。



## 语法和字符集

标签是键值对，合法的键包括两个部分：可选的前缀和名字，前缀和名字使用`/` 分割。名字部分的长度不能超过63个字符，字母,数字（ `a-z0-9A-Z`），`-`，`_`，`.`都是合法字符 。kubernetes.io/ and k8s.io/前缀是留给Kubernetes核心组件使用。



## 标签选择器

Unlike names不像名字和UID，标签不提供唯一性。一般来说，我们期望多个对象拥有相同的标签。

通过标签选择器，客户端和用户可以识别出一组对象。标签选择器是Kubernetes中的核心分组原语。

The API currently supports two types of selectors: equality-based and set-based. A label selector can be made of multiple requirements which are comma-separated. In the case of multiple requirements, all must be satisfied so the comma separator acts as a logical AND (&&) operator.

The semantics of empty or non-specified selectors are dependent on the context, and API types that use selectors should document the validity and meaning of them.

Note: For some API types, such as ReplicaSets, the label selectors of two instances must not overlap within a namespace, or the controller can see that as conflicting instructions and fail to determine how many replicas should be present.

Equality-based requirement

Equality- or inequality-based requirements allow filtering by label keys and values. Matching objects must satisfy all of the specified label constraints, though they may have additional labels as well. Three kinds of operators are admitted =,==,!=. The first two represent equality (and are simply synonyms), while the latter represents inequality. For example:

    environment = production
    tier != frontend

The former selects all resources with key equal to environment and value equal to production. The latter selects all resources with key equal to tier and value distinct from frontend, and all resources with no labels with the tier key. One could filter for resources in production excluding frontend using the comma operator: environment=production,tier!=frontend

One usage scenario for equality-based label requirement is for Pods to specify node selection criteria. For example, the sample Pod below selects nodes with the label “accelerator=nvidia-tesla-p100”.

    apiVersion: v1
    kind: Pod
    metadata:
      name: cuda-test
    spec:
      containers:
        - name: cuda-test
          image: "k8s.gcr.io/cuda-vector-add:v0.1"
          resources:
            limits:
              nvidia.com/gpu: 1
      nodeSelector:
        accelerator: nvidia-tesla-p100

Set-based requirement

Set-based label requirements allow filtering keys according to a set of values. Three kinds of operators are supported: in,notin and exists (only the key identifier). For example:

    environment in (production, qa)
    tier notin (frontend, backend)
    partition
    !partition

The first example selects all resources with key equal to environment and value equal to production or qa. The second example selects all resources with key equal to tier and values other than frontend and backend, and all resources with no labels with the tier key. The third example selects all resources including a label with key partition; no values are checked. The fourth example selects all resources without a label with key partition; no values are checked. Similarly the comma separator acts as an AND operator. So filtering resources with a partition key (no matter the value) and with environment different than  qa can be achieved using partition,environment notin (qa). The set-based label selector is a general form of equality since environment=production is equivalent to environment in (production); similarly for != and notin.

Set-based requirements can be mixed with equality-based requirements. For example: partition in (customerA, customerB),environment!=qa.

## API

LIST and WATCH filtering

LIST and WATCH operations may specify label selectors to filter the sets of objects returned using a query parameter. Both requirements are permitted (presented here as they would appear in a URL query string):

- equality-based requirements: ?labelSelector=environment%3Dproduction,tier%3Dfrontend
- set-based requirements: ?labelSelector=environment+in+%28production%2Cqa%29%2Ctier+in+%28frontend%29

Both label selector styles can be used to list or watch resources via a REST client. For example, targeting apiserver with kubectl and using equality-based one may write:

    kubectl get pods -l environment=production,tier=frontend

or using set-based requirements:

    kubectl get pods -l 'environment in (production),tier in (frontend)'

As already mentioned set-based requirements are more expressive.  For instance, they can implement the OR operator on values:

    kubectl get pods -l 'environment in (production, qa)'

or restricting negative matching via exists operator:

    kubectl get pods -l 'environment,environment notin (frontend)'

Set references in API objects

Some Kubernetes objects, such as services and replicationcontrollers, also use label selectors to specify sets of other resources, such as pods.

Service and ReplicationController

The set of pods that a service targets is defined with a label selector. Similarly, the population of pods that a replicationcontroller should manage is also defined with a label selector.

Labels selectors for both objects are defined in json or yaml files using maps, and only equality-based requirement selectors are supported:

    "selector": {
        "component" : "redis",
    }

or

    selector:
        component: redis

this selector (respectively in json or yaml format) is equivalent to component=redis or component in (redis).

Resources that support set-based requirements

Newer resources, such as Job, Deployment, Replica Set, and Daemon Set, support set-based requirements as well.

    selector:
      matchLabels:
        component: redis
      matchExpressions:
        - {key: tier, operator: In, values: [cache]}
        - {key: environment, operator: NotIn, values: [dev]}

matchLabels is a map of {key,value} pairs. A single {key,value} in the matchLabels map is equivalent to an element of matchExpressions, whose key field is “key”, the operator is “In”, and the values array contains only “value”. matchExpressions is a list of pod selector requirements. Valid operators include In, NotIn, Exists, and DoesNotExist. The values set must be non-empty in the case of In and NotIn. All of the requirements, from both matchLabels and matchExpressions are ANDed together – they must all be satisfied in order to match.

Selecting sets of nodes

One use case for selecting over labels is to constrain the set of nodes onto which a pod can schedule. See the documentation on node selection for more information.
