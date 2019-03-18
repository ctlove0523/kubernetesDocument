# Names

Kubernetes REST API 中所有的对象都由Name和UID唯一标识。对于用户提供的非唯一属性，Kubernetes提供labels和annotations。

对于Names和UIDs精确的语法规则，可以参考[设计文档](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/architecture/identifiers.md)



## Names

客户端提供一个字符串，用于引用URL中的资源对象，比如`/api/v1/pods/some-name` 。

一个给定类型的对象一次只能由一个名称。但是，如果你删除了这个对象，你可以新建一个具有相同名字的对象。

按照惯例，Kubernetes资源的名字不得超过253个字符，只能由小写字符、数字、-和.组成，一些特殊的资源对名字的要求可能更加严格。

## UIDs

UIDs是Kubernetes生成用于唯一表示对象的字符串。

Kubernetes创建的对象在整个生命周期过程中都由一个唯一的UID。它旨在区分类似的历史实体。

