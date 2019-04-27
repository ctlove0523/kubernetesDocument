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

> 注意：Pod Preset能够在适当的时候修改Pod spec中的`.spec.containers`  字段。Pod Preset中的资源定义不会应用于`initContainers` 字段。

### 禁用特定Pod的Pod Preset

在某些情况下，你希望Pod不会被任何Pod Preset改变。在这些场景，你可以在Pod Spec中添加如下格式的注释：`podpreset.admission.kubernetes.io/exclude: "true"`。

## 启用 Pod Preset

要在群集中使用Pod Presets，您必须确保以下内容：

1. 已经启用 `settings.k8s.io/v1alpha1/podpreset` 类型API。
2. 已启用准入控制器`PodPreset` 。
3. 已经在使用的命名空间内创建`PodPreset` 对象来定义Presets。