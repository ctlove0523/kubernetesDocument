# Names

~~All objects in the Kubernetes REST API are unambiguously identified by a Name and a UID.~~

~~For non-unique user-provided attributes, Kubernetes provides [labels](https://kubernetes.io/docs/user-guide/labels) and [annotations](https://kubernetes.io/docs/concepts/overview/working-with-objects/annotations/).~~

~~See the [identifiers design doc](https://git.k8s.io/community/contributors/design-proposals/architecture/identifiers.md) for the precise syntax rules for Names and UIDs.~~

Kubernetes REST API 中所有的对象都由Name和UID唯一标识。对于用户提供的非唯一属性，Kubernetes提供labels和annotations。

对于Names和UIDs精确的语法规则，可以参考[设计文档](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/architecture/identifiers.md)



## Names

~~A client-provided string that refers to an object in a resource URL, such as `/api/v1/pods/some-name`.~~

~~Only one object of a given kind can have a given name at a time. However, if you delete the object, you can make a new object with the same name.~~

~~By convention, the names of Kubernetes resources should be up to maximum length of 253 characters and consist of lower case alphanumeric characters, `-`, and `.`, but certain resources have more specific restrictions.~~

客户端提供一个字符串，用于引用URL中的资源对象，比如`/api/v1/pods/some-name` 。

一个给定类型的对象一次只能由一个名称。但是，如果你删除了这个对象，你可以新建一个具有相同名字的对象。

按照惯例，Kubernetes资源的名字不得超过253个字符，只能由小写字符、数字、-和.组成，一些特殊的资源对名字的要求可能更加严格。

## UIDs

~~A Kubernetes systems-generated string to uniquely identify objects.~~

~~Every object created over the whole lifetime of a Kubernetes cluster has a distinct UID. It is intended to distinguish between historical occurrences of similar entities.~~

UIDs是Kubernetes生成用于唯一表示对象的字符串。

Kubernetes创建的对象在整个生命周期过程中都由一个唯一的UID。它旨在区分类似的历史实体。

