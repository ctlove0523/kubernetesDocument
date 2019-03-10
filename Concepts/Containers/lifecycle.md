# 容器生命周期钩子

## 概要

和许多具有生命周期钩子组件的编程语义框架一样，kubernetes为容器提供了生命周期钩子。钩子让容器可以了解其管理的生命周期中发生的事件，在执行相应的生命周期钩子时运行在处理程序中实现的代码，这和回调函数的工作方法类似。

## 容器钩子

容器中暴露了两个钩子：

```
PostStart
```

这个钩子在容器创建后立即被执行，但是不保证在容器`ENTRYPOINT`之前执行这个钩子。不会向处理器传递任何参数。

```
PreStop
```

这个钩子在API 请求或管理事件（存活探针检查失败，抢占，资源争夺等）导致的容器停止之前立即被调用。如果容器的状态为已停止或已完成，preStop钩子的调用会失败。钩子的调用为阻塞式（同步），所以在发送删除容器命令前，钩子执行必须结束。同样不会向处理器传递任何参数。

### 钩子处理器实现

容器可以通过实现和注册该钩子的处理器来访问钩子。可以为容器实现两种类型的钩子处理器：

* Exec - 在容器的cgroups和命名空间内执行特定命令，例如pre-stop.sh。
* HTTP - 对Container上的特定端点执行HTTP请求。

### 执行钩子处理器

当容器生命周期钩子被调用时，Kubernetes管理系统会执行容器内向该钩子注册的处理器。

~~Hook handler calls are synchronous within the context of the Pod containing the Container. This means that for a `PostStart`hook, the Container ENTRYPOINT and hook fire asynchronously. However, if the hook takes too long to run or hangs, the Container cannot reach a `running` state.~~

在包含容器Pod的上下文中，钩子处理的调用是同步的。这意味着对`postStart` 钩子，容器 `ENTRYPOINT` 和钩子调用是异步的。然而，如果钩子需要很长的执行事件或被挂住，容器就不能处于 `running` 状态。 

~~The behavior is similar for a `PreStop` hook. If the hook hangs during execution, the Pod phase stays in a `Terminating` state and is killed after `terminationGracePeriodSeconds` of pod ends. If a `PostStart` or `PreStop` hook fails, it kills the Container.~~

`PreStop` 钩子的行为与 `PostStart` 类似。如果，钩子在执行期间被挂住，Pod的状态将一直保留在 `Terminating` 状态，pod终止后经过  `terminationGracePeriodSeconds`  时间后容器将被杀死。如果 `PostStart`或 `PreStop` 执行失败，容器也会被杀掉。 

~~Users should make their hook handlers as lightweight as possible. There are cases, however, when long running commands make sense, such as when saving state prior to stopping a Container.~~

用户应当将钩子处理器设计的尽可能简单。但是，在有些情况下，长时间执行的命令是有用的，比如在容器停止之前保存状态。

### 钩子传递保证

~~Hook delivery is intended to be *at least once*, which means that a hook may be called multiple times for any given event, such as for `PostStart` or `PreStop`. It is up to the hook implementation to handle this correctly.~~

钩子的传递保证至少一次，这意味着对于给定的事件，钩子可能被执行多次，比如  `PostStart` 或 `PreStop` 。钩子实现需要处理这种情况。

~~Generally, only single deliveries are made. If, for example, an HTTP hook receiver is down and is unable to take traffic, there is no attempt to resend. In some rare cases, however, double delivery may occur. For instance, if a kubelet restarts in the middle of sending a hook, the hook might be resent after the kubelet comes back up.~~

通常，需要仅传递一次。例如，如果一个HTTP钩子接收器已经关闭而且没有任何流量，则不会尝试重新发送。在一些少见的情况下，可能出现两次传递。比如，在钩子传递过程中kubelet重启，当kubelet重启完成后，钩子可能被再次传递。

### 调试钩子处理器

~~The logs for a Hook handler are not exposed in Pod events. If a handler fails for some reason, it broadcasts an event. For `PostStart`, this is the `FailedPostStartHook` event, and for `PreStop`, this is the `FailedPreStopHook` event. You can see these events by running `kubectl describe pod <pod_name>`. Here is some example output of events from running this command:~~

在Pod的事件中不会包括钩子处理器日志。如果因为某些原因处理器执行失败，对应的事件将会被广播。对于 `PostStart` 会产生一个 `FailedPostStartHook` 事件，`PreStop` 会产生一个 `FailedPreStopHook`  事件。通过 `kubectl describe pod <pod_name>` 可以看到对应的事件。下面是一个样例的输出：

```
Events:
  FirstSeen  LastSeen  Count  From                                                   SubobjectPath          Type      Reason               Message
  ---------  --------  -----  ----                                                   -------------          --------  ------               -------
  1m         1m        1      {default-scheduler }                                                          Normal    Scheduled            Successfully assigned test-1730497541-cq1d2 to gke-test-cluster-default-pool-a07e5d30-siqd
  1m         1m        1      {kubelet gke-test-cluster-default-pool-a07e5d30-siqd}  spec.containers{main}  Normal    Pulling              pulling image "test:1.0"
  1m         1m        1      {kubelet gke-test-cluster-default-pool-a07e5d30-siqd}  spec.containers{main}  Normal    Created              Created container with docker id 5c6a256a2567; Security:[seccomp=unconfined]
  1m         1m        1      {kubelet gke-test-cluster-default-pool-a07e5d30-siqd}  spec.containers{main}  Normal    Pulled               Successfully pulled image "test:1.0"
  1m         1m        1      {kubelet gke-test-cluster-default-pool-a07e5d30-siqd}  spec.containers{main}  Normal    Started              Started container with docker id 5c6a256a2567
  38s        38s       1      {kubelet gke-test-cluster-default-pool-a07e5d30-siqd}  spec.containers{main}  Normal    Killing              Killing container with docker id 5c6a256a2567: PostStart handler: Error executing in Docker Container: 1
  37s        37s       1      {kubelet gke-test-cluster-default-pool-a07e5d30-siqd}  spec.containers{main}  Normal    Killing              Killing container with docker id 8df9fdfd7054: PostStart handler: Error executing in Docker Container: 1
  38s        37s       2      {kubelet gke-test-cluster-default-pool-a07e5d30-siqd}                         Warning   FailedSync           Error syncing pod, skipping: failed to "StartContainer" for "main" with RunContainerError: "PostStart handler: Error executing in Docker Container: 1"
  1m         22s       2      {kubelet gke-test-cluster-default-pool-a07e5d30-siqd}  spec.containers{main}  Warning   FailedPostStartHook
```

