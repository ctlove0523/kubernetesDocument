# Pod Preset

本章将对Pod Presets进行简单的介绍，Pod Presets是用于在特定的时间向Pod注入特定信息的对象。注入的信息包括密钥，卷，卷挂载和环境变量。

## 理解 Pod Presets

`Pod Preset` 是在特定时间向Pod注入额外运行时要求的一种API 资源。可以使用`label selectors` 选择Pod Preset需要注入的Pod。

使用Pod Preset 允许pod 模板的作者不用必须显示的给每一个pod提供所有的信息。这样，使用特定服务的pod模板的作者不需要知道有关该服务的所有详细信息。

有关Pod Preset的背景信息, 参考 [design proposal for PodPreset](https://git.k8s.io/community/contributors/design-proposals/service-catalog/pod-preset.md).

## Pod Preset如何工作

Kubernetes提供了一个准入控制器（`PodPreset`），启用该控制器后，创建Pod的请求将会应用Pod Preset。当出现一个pod创建请求时，系统做以下动作：

1. 检索可供使用的所有`PodPresets` 。
2. 检查任意一个`PodPreset` 的 标签选择器是否匹配被创建pod的标签。
3. 尝试将`PodPreset` 定义的不同资源合并到被创建的Pod中。
4. 对于错误，抛出一个事件文档描述在pod上合并错误，然后不适用任何`PodPreset` 注入的资源创建Pod。
5. 注释生成的修改后的Pod规范，以指示它已被PodPreset修改。注释的形式为： `podpreset.admission.kubernetes.io/podpreset-<pod-preset name>: "<resource version>"`。

每个Pod可以匹配0个或多个Pod Preset，每个`PodPreset` 可以用于0个或多个pod。当一个`PodPreset` 应用于一个或多个Pod时，Kubernetes修改Pod Spec。为了修改`Env`, `EnvFrom`, 和 `VolumeMounts` kubernetes需要修改Pod内所有容器的spec；为了修改 `Volume`,kubernetes修改Pod的Spec。

> **Note:** A Pod Preset is capable of modifying the `.spec.containers` field in a Pod spec when appropriate. *No* resource definition from the Pod Preset will be applied to the `initContainers` field.

### Disable Pod Preset for a Specific Pod

There may be instances where you wish for a Pod to not be altered by any Pod Preset mutations. In these cases, you can add an annotation in the Pod Spec of the form: `podpreset.admission.kubernetes.io/exclude: "true"`.

## 启用 Pod Preset

In order to use Pod Presets in your cluster you must ensure the following:

1. You have enabled the API type `settings.k8s.io/v1alpha1/podpreset`. For example, this can be done by including `settings.k8s.io/v1alpha1=true` in the `--runtime-config` option for the API server. In minikube add this flag `--extra-config=apiserver.runtime-config=settings.k8s.io/v1alpha1=true` while starting the cluster.
2. You have enabled the admission controller `PodPreset`. One way to doing this is to include `PodPreset` in the `--enable-admission-plugins` option value specified for the API server. In minikube add this flag `--extra-config=apiserver.enable-admission-plugins=Initializers,NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,DefaultTolerationSeconds,NodeRestriction,MutatingAdmissionWebhook,ValidatingAdmissionWebhook,ResourceQuota,PodPreset`while starting the cluster.
3. You have defined your Pod Presets by creating `PodPreset` objects in the namespace you will use.