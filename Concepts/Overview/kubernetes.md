# Kubernetes是什么？

~~This page is an overview of Kubernetes~~

本文分以下几个主题简单介绍以下Kubernetes。

- 为什么需要Kubernetes，Kubernetes能干什么？
- Kubernetes如何成为一个平台？
- Kubernetes不是什么？
- 为什么是容器？
- Kubernetes和K8s是什么含义？

Kubernetes is a portable, extensible open-source platform for managing containerized workloads and services, that facilitates both declarative configuration and automation. It has a large, rapidly growing ecosystem. Kubernetes services, support, and tools are widely available.

Google open-sourced the Kubernetes project in 2014. Kubernetes builds upon a [decade and a half of experience that Google has with running production workloads at scale](https://research.google.com/pubs/pub43438.html), combined with best-of-breed ideas and practices from the community.

Kubernetes是一个用于管理容器化的工作负载和服务,可移植，可扩展的开源平台，对声明性配置和自动化有很好的支持。Kubernetes拥有庞大，快速发展的生态系统。 Kubernetes服务，支持和工具得到广泛应用。

谷歌在2014年开源了Kubernetes项目.Kubernetes建立在谷歌大规模运行生产负载十五年经验的基础上，并结合了社区中的最佳创意和实践。


## 为什么需要Kubernetes，Kubernetes能干什么？

~~Kubernetes has a number of features. It can be thought of as:~~

- ~~a container platform~~
- ~~a microservices platform~~
- ~~a portable cloud platform and a lot more.~~

~~Kubernetes provides a **container-centric** management environment. It orchestrates computing, networking, and storage infrastructure on behalf of user workloads. This provides much of the simplicity of Platform as a Service (PaaS) with the flexibility of Infrastructure as a Service (IaaS), and enables portability across infrastructure providers.~~

Kubernetes有很多特性，可以把Kubernetes定义为以下平台：

* 容器平台
* 微服务平台
* 便携式的云平台等等。

Kubernetes提供以容器为中心的管理环境。它代表客户的工作负载对计算、网络和存储架构进行协调。Kubernetes具备PaaS平台的简单性，IaaS 平台的灵活性，并支持跨基础架构提供商的可移植性。

## Kubernetes如何成为一个平台

~~Even though Kubernetes provides a lot of functionality, there are always new scenarios that would benefit from new features. Application-specific workflows can be streamlined to accelerate developer velocity. Ad hoc orchestration that is acceptable initially often requires robust automation at scale. This is why Kubernetes was also designed to serve as a platform for building an ecosystem of components and tools to make it easier to deploy, scale, and manage applications.~~

~~[Labels](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/) empower users to organize their resources however they please. [Annotations](https://kubernetes.io/docs/concepts/overview/working-with-objects/annotations/) enable users to decorate resources with custom information to facilitate their workflows and provide an easy way for management tools to checkpoint state.~~

~~Additionally, the [Kubernetes control plane](https://kubernetes.io/docs/concepts/overview/components/) is built upon the same [APIs](https://kubernetes.io/docs/reference/using-api/api-overview/) that are available to developers and users. Users can write their own controllers, such as [schedulers](https://github.com/kubernetes/community/blob/master/contributors/devel/scheduler.md), with [their own APIs](https://kubernetes.io/docs/concepts/api-extension/custom-resources/) that can be targeted by a general-purpose [command-line tool](https://kubernetes.io/docs/user-guide/kubectl-overview/).~~

~~This [design](https://git.k8s.io/community/contributors/design-proposals/architecture/architecture.md) has enabled a number of other systems to build atop Kubernetes.~~

尽管Kubernetes提供了许多功能，但总会有新的方案可以从新功能中受益。可以简化特定于应用程序的工作流程，以加快开发人员的速度。当初可以接受的临时编排往往需要稳定的大规模自动化。这就是Kubernetes为什么还可以作为构件和工具生态系统的平台，Kubernetes使得部署，扩展以及应用管理变得更加简单。

标签使得用户可以随意组合资源。Annotations使用户可以使用用户自定义信息修饰资源，以方便用户自己的工作流并未管理工具提供检查点状态的简单方法。

此外，Kubernetes管里面使用和开发者和用户一致的API构建。用户可以自己编写控制器（调度器等），这些自定义的API可以通过命令行工具使用。



## Kubernetes不是什么？

~~Kubernetes is not a traditional, all-inclusive PaaS (Platform as a Service) system. Since Kubernetes operates at the container level rather than at the hardware level, it provides some generally applicable features common to PaaS offerings, such as deployment, scaling, load balancing, logging, and monitoring. However, Kubernetes is not monolithic, and these default solutions are optional and pluggable. Kubernetes provides the building blocks for building developer platforms, but preserves user choice and flexibility where it is important.~~

Kubernetes不是一个传统的，包括万象的PaaS系统。由于，Kubernetes在容器级而不是硬件级运行，Kubernetes提供了一些PaaS平台通用的功能，比如部署、扩展、负载均衡、日志和监控。但是Kubernetes不是单一的，而且这些PaaS平台默认的功能是可选可插拔的。Kubernetes为构建开发者平台提供了基础构件，但在重要的地方保留了用户的可选择性以及灵活性。

Kubernetes:

- ~~Does not limit the types of applications supported. Kubernetes aims to support an extremely diverse variety of workloads, including stateless, stateful, and data-processing workloads. If an application can run in a container, it should run great on Kubernetes.~~
- 支持的应用类型不受限制。Kubernetes的目标是支持各种各样的工作负载，包括无状态工作负载，有状态负载和数据处理工作负载等。如果一个应用可以在容器内运行，那么该应用可以很好的在Kubernetes上运行。
- ~~Does not deploy source code and does not build your application. Continuous Integration, Delivery, and Deployment (CI/CD) workflows are determined by organization cultures and preferences as well as technical requirements.~~
- 不部署源代码也不为用户构建应用。持续集成，交付和部署（CI / CD）的工作流程由组织文化和偏好以及技术要求决定。
- ~~Does not provide application-level services, such as middleware (e.g., message buses), data-processing frameworks (for example, Spark), databases (e.g., mysql), caches, nor cluster storage systems (e.g., Ceph) as built-in services. Such components can run on Kubernetes, and/or can be accessed by applications running on Kubernetes through portable mechanisms, such as the Open Service Broker.~~
- 不提供应用级的服务，像中间件（消息总系），数据处理框架（Spark），数据库（MySQL），缓存，集群存储系统（Ceph）不会作为内嵌服务。这些组件可以运行在Kubernetes上，运行在Kubernetes可以通过合适的机制访问这些组件，比如通过开放式服务代理。
- ~~Does not dictate logging, monitoring, or alerting solutions. It provides some integrations as proof of concept, and mechanisms to collect and export metrics.~~
- 不是日志，监控，告警解决方案。Kubernetes提供集成方法和机器来收集并导出指标。
- ~~Does not provide nor mandate a configuration language/system (e.g., [jsonnet](https://github.com/google/jsonnet)). It provides a declarative API that may be targeted by arbitrary forms of declarative specifications.~~
- 不提供和授权配置语言和配置系统。Kubernetes 提供了一个声明性API，并可以通过任意形式的声明性规范来实现。
- ~~Does not provide nor adopt any comprehensive machine configuration, maintenance, management, or self-healing systems.~~
- 不提供或采用任何全面的机器配置，维护，管理或自我修复系统。

~~Additionally, Kubernetes is not a mere *orchestration system*. In fact, it eliminates the need for orchestration. The technical definition of *orchestration* is execution of a defined workflow: first do A, then B, then C. In contrast, Kubernetes is comprised of a set of independent, composable control processes that continuously drive the current state towards the provided desired state. It shouldn’t matter how you get from A to C. Centralized control is also not required. This results in a system that is easier to use and more powerful, robust, resilient, and extensible.~~

此外，Kubernetes 不仅仅是一个编排系统。事实上，Kubernetes消除了对编排的需要。编排的定义是执行已定义的工作流程：A → B → C。相比之下，Kubernetes由一组独立的，可组合的控制过程组成，这些过程不断地将当前状态推向所提供的所需状态。从状态A到状态C的方法无关紧要，也不再需要集中控制。这使得系统更加易用，更加强大，更加鲁棒，具有更强的弹性和可扩展性。

## 为什么是容器

~~Looking for reasons why you should be using containers?~~

为社么需要使用容器？

![Why Containers?](https://d33wubrfki0l68.cloudfront.net/e7b766e0175f30ae37f7e0e349b87cfe2034a1ae/3e391/images/docs/why_containers.svg)

The *Old Way* to deploy applications was to install the applications on a host using the operating-system package manager. This had the disadvantage of entangling the applications’ executables, configuration, libraries, and lifecycles with each other and with the host OS. One could build immutable virtual-machine images in order to achieve predictable rollouts and rollbacks, but VMs are heavyweight and non-portable.

The *New Way* is to deploy containers based on operating-system-level virtualization rather than hardware virtualization. These containers are isolated from each other and from the host: they have their own filesystems, they can’t see each others’ processes, and their computational resource usage can be bounded. They are easier to build than VMs, and because they are decoupled from the underlying infrastructure and from the host filesystem, they are portable across clouds and OS distributions.

Because containers are small and fast, one application can be packed in each container image. This one-to-one application-to-image relationship unlocks the full benefits of containers. With containers, immutable container images can be created at build/release time rather than deployment time, since each application doesn’t need to be composed with the rest of the application stack, nor married to the production infrastructure environment. Generating container images at build/release time enables a consistent environment to be carried from development into production. Similarly, containers are vastly more transparent than VMs, which facilitates monitoring and management. This is especially true when the containers’ process lifecycles are managed by the infrastructure rather than hidden by a process supervisor inside the container. Finally, with a single application per container, managing the containers becomes tantamount to managing deployment of the application.

Summary of container benefits:

- **Agile application creation and deployment**: Increased ease and efficiency of container image creation compared to VM image use.
- **Continuous development, integration, and deployment**: Provides for reliable and frequent container image build and deployment with quick and easy rollbacks (due to image immutability).
- **Dev and Ops separation of concerns**: Create application container images at build/release time rather than deployment time, thereby decoupling applications from infrastructure.
- **Observability** Not only surfaces OS-level information and metrics, but also application health and other signals.
- **Environmental consistency across development, testing, and production**: Runs the same on a laptop as it does in the cloud.
- **Cloud and OS distribution portability**: Runs on Ubuntu, RHEL, CoreOS, on-prem, Google Kubernetes Engine, and anywhere else.
- **Application-centric management**: Raises the level of abstraction from running an OS on virtual hardware to running an application on an OS using logical resources.
- **Loosely coupled, distributed, elastic, liberated micro-services**: Applications are broken into smaller, independent pieces and can be deployed and managed dynamically – not a monolithic stack running on one big single-purpose machine.
- **Resource isolation**: Predictable application performance.
- **Resource utilization**: High efficiency and density.

老的部署应用的方法是使用操作系统的包管理器在主机上部署应用。采用这种方式部署应用的缺点是，应用的执行文件、配置、库和生命周期互相耦合，而且都与主机的操作系统耦合。为了实现可预测的部署和回滚，可以采用构建不可变虚拟机镜像的方法，但是虚拟机非常重而且不可移植。

新的部署方法基于操作系统级别虚拟计划而非硬件级别虚拟化来部署容器。这些容器间彼此隔离而且与主机隔离：容器拥有自己的文件系统，无法看到彼此的进程，容器使用的计算资源也受到限制。容器比虚拟机更加容易构建，容器与底层的基础设施和主机文件系统分离，因此容器可以跨云和操作系统分发。

因为容器更加轻量更快，每个应用程序可以打包到各自的镜像中。这种应用与镜像一对一的关系释放了容器所有的优势。使用容器，可以在构建/发布时而不是部署是创建不可变的容器镜像，因为每个应用程序不需要和应用堆栈中的其他组件以及生产环境相组合。在构建/发布时制作镜像可以保证开发环境和生产环境的一致性。同样，容器比虚拟机更加透明，这有利于监控和管理。当容器内进程的声明周期由基础设施而不是隐藏在容器内的管理器管理时，容器的优势更加明显。最后，一个应用一个容器，管理容器和管理应用部署几乎等价。

容器益处总结：

- 应用创建和部署更加敏捷。和VM镜像相比，容器镜像的构建更加简单高效。
- 持续开发、集成和部署。
- 开发运维分离。
- 可监控性。
- 开发、测试、部署和生产环境一致性。
- 跨云和OS移植。
- 以应用为中心的管理。
- 松耦合、分布式、弹性和微服务。
- 资源隔离。
- 资源高利用率。


## Kubernetes和K8s是什么含义？

~~The name **Kubernetes** originates from Greek, meaning *helmsman* or *pilot*, and is the root of *governor* and [cybernetic](http://www.etymonline.com/index.php?term=cybernetics). *K8s* is an abbreviation derived by replacing the 8 letters “ubernete” with “8”.~~

Kubernetes这个名字来源自希腊语，意思是舵手或飞行员。K8S是简称，因为“ubernete”刚好有8个字符。
