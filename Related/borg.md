# Large-scale cluster management at Google with Borg

## 摘要

Google's Borg system is a cluster manager that runs hundreds of thousands of jobs，from many thousands of different applications, across a number of clusters each with up to tens of thousands of machines.

In achieves high utilization by combining admission control,efficient task-packing,over-commitment,and machine sharing with process-level performance isolation.It supports high-availability applications with runtime features that minimize fault-recovery time,and scheduling policies that reduce the probability of correlated failures. Borg simplifies life for it user by offering a declarative job specification language, name service integration, real-time job monitoring, and tools to analyze and simulate system behavior.

We present a summary of the Borg system architecture and features,important design decision,q quantitative analysis of some of its policy decision,and a qualitative examination of lessons learned from a decade of operational experience with it.

Google的Borg系统是一个集群管理器，可以运行来自数千个不同的应用程序中的数十万个作业，这些作业分布在多个集群中，每个集群最多有数万台机器。

Borg通过准入控制，高效的任务打包，over-commitment和进程及性能隔离的机制共享实现更高的利用率。Borg通过最大限度的缩短故障恢复时间，通过调度降低级联故障的概率来为具有运行时功能的应用提供搞可用性。Borg通过提供声明性作业规范语言，集成名称服务，实时作业监控以及分析和模拟系统行为的工具，简化了用户的工作。

在本文中我们重点概述Borg系统的架构和特性，重要设计决策，部分决策的定量分析，以及运营Borg数十年的经验教训。

### 1. 引言

The cluster management system we internally call Borg admits,schedules, starts, restarts, and monitors the full range of applications that Google runs. This paper explains how.

Borg provides three main benefits: it (1) hides the details of resource management and failure handling so its users can focus on application development instead; (2) operates with very high reliability and availability, and supports applications that do the same; and (3) lets us run workloads across tens of thousands of machines effectively. Borg is not the first system to address these issues, but it’s one of the few operating at this scale, with this degree of resiliency and completeness. This paper is organized around these topics, concluding with a set of qualitative observations we have made from operating Borg in production for more than a decade.



Google内部称为Borg的集群管理系统负责Google内部允许的所有应用的admits、调度、启动、重启和监控。在这篇论文中我们将解释Brog如何做到的。

Borg主要提供三个益处：

1. 屏蔽资源管理和错误处理的细节，让用户可以专注应用的开发。
2. 自身具有高可用高可靠，同时支持应用的高可用和高可靠。
3. 支持应用跨数前台机器高效运行。

Borg不是第一个解决这些问题的系统，但是Borg是少数几个以这种规模运行的系统之一，拥有相当高的韧性和完整性。本文围绕这些主题阻止，包括Borg在生产环境十多年来所做的一系列定性观察。

## 2. 用户视角

Borg’s users are Google developers and system administrators (site reliability engineers or SREs) that run Google’s applications and services. Users submit their work to Borg in the form of jobs, each of which consists of one or more tasks that all run the same program (binary). Each job runs in one Borg cell, a set of machines that are managed as a unit. The remainder of this section describes the main features exposed in the user view of Borg.

Borg的用户是运行Google应用和服务的Google开发人员和系统管理员（站点可靠性工程师或者SRE）。用户以job的形式向Borg提供自己的工作，每个job都包含一个或多个运行在相同程序（二进制）的任务。每个job运行在Borg的一个cell中，一个cell包含一组机器，cell是Borg管理资源的基本单位。本章的其余部分将以用户视角介绍Borg的主要功能。

### 2.1 工作负载

Borg cells run a heterogenous workload with two main parts. The first is long-running services that should “never” go down, and handle short-lived latency-sensitive requests (a few µs to a few hundred ms). Such services are used for end-user-facing products such as Gmail, Google Docs, and web search, and for internal infrastructure services (e.g., BigTable). The second is batch jobs that take from a few seconds to a few days to complete; these are much less sensitive to short-term performance fluctuations. The workload
mix varies across cells, which run different mixes of applications depending on their major tenants (e.g., some cells are quite batch-intensive), and also varies over time: batch jobs come and go, and many end-user-facing service jobs see a diurnal usage pattern. Borg is required to handle all these cases equally well.

Borg cell运行的异质工作负载主要有两部分。第一部分是长时间运行的服务，这些服务永远不能下线，并处理对低延迟敏感的请求（几us到几百ms）。这些服务用于面向用户的产品（例如,Gmail，Google Docs，Web搜索）和内部基础设施服务（比如，BigTable）。第二部分是批量jobs，这些jobs完成需要的时间从几秒到几天，这些jobs对短期的性能波动不敏感。每个cell运行不同的工作负载，根据该cell的主要租户运行不同的应用组合（比如有些cell主要运行批量job），不同时间运行的负载也不一样：批量jobs来去匆匆，许多面向终端用户的job也有昼夜模式。Borg必须同样的处理这些所有的场景。

A representative Borg workload can be found in a publicly-available month-long trace from May 2011 [80], which has been extensively analyzed (e.g., [68] and [1, 26, 27, 57]).

> 不是阐述Borg的一部分，省略翻译。

Many application frameworks have been built on top of Borg over the last few years, including our internal MapReduce system [23], FlumeJava [18], Millwheel [3], and Pregel [59]. Most of these have a controller that submits a master job and one or more worker jobs; the first two play a similar role to YARN’s application manager [76]. Our distributed storage systems such as GFS [34] and its successor CFS, Bigtable [19], and Megastore [8] all run on Borg.

在过去的几年中，许多应用程序框架都是构建于Borg之上，包括Google内部的MapReduce系统，FlumeJava，Millwheel和Pregel。其中大多数框架都有提交一个主要job和一个或多个worker job的控制器；前两个应用框架和YARN的应用管理器相似。Google的分布式存储系统，比如GFS和后继CFS，Bigtable和Megastor都运行在Borg之上。

For this paper, we classify higher-priority Borg jobs as “production” (prod) ones, and the rest as “non-production”(non-prod). Most long-running server jobs are prod; most batch jobs are non-prod. In a representative cell, prod jobs are allocated about 70% of the total CPU resources and represent
about 60% of the total CPU usage; they are allocated about 55% of the total memory and represent about 85% of the total memory usage. The discrepancies between allocation and usage will prove important in §5.5.

在本文中，将优先级高的Borg 作业归类为“生产”作业（prod），其余的归类为“非生产”作业（non-prod）。大多数长时间运行的作业是生产作业，大多数的批量作业是非生产作业。在具有代表性的cell中，生产作业被分配了70%的CPU资源，并占总CPU使用了的60%；分配了55%的内存，内存使用占总的内存使用的85%。分配和使用之间的差异将在5.5中证明。

### 2.2 集群 和 cells

The machines in a cell belong to a single cluster, defined by the high-performance datacenter-scale network fabric that connects them. A cluster lives inside a single datacenter building, and a collection of buildings makes up a site. A cluster usually hosts one large cell and may have a few smaller-scale test or special-purpose cells. We assiduously avoid any single point of failure.

cell中的机器属于单个集群，由连接机器的高性能数据中心规模的网络结构定义。一个集群只存在于单个数据中心中，多个数据中心构成一个站点。一个集群通常承载一个大型cell，并且可能具有一些较小规模的测试或特殊用途的cell。Borg努力的避免任何单点故障。

Our median cell size is about 10 k machines after excluding test cells; some are much larger. The machines in a cell are heterogeneous in many dimensions: sizes (CPU, RAM,disk, network), processor type, performance, and capabilities such as an external IP address or flash storage. Borg isolates
users from most of these differences by determining where in a cell to run tasks, allocating their resources, installing their programs and other dependencies, monitoring their health, and restarting them if they fail.

Borg中型的cell一般包括10,000台左右的机器（排除测试cell），有些cell的规模更大。cell中的机器从多个维度来看都是异构的：大小（CPU，内存，磁盘，网络），处理器类型，性能和能力，比如是否有EIP，是否有闪存。Borg通过确定cell中运行任务的位置，分配资源，安装程序和其他依赖项，监视其运行状况以及在失败时重新启动来将用户与大多数差异隔离开来。

### 2.3 Jobs and tasks

A Borg job’s properties include its name, owner, and the number of tasks it has. Jobs can have constraints to force its tasks to run on machines with particular attributes such as processor architecture, OS version, or an external IP address. Constraints can be hard or soft; the latter act like preferences rather than requirements. The start of a job can be deferred until a prior one finishes. A job runs in just one cell.



Each task maps to a set of Linux processes running in a container on a machine [62]. The vast majority of the Borg workload does not run inside virtual machines (VMs),because we don’t want to pay the cost of virtualization. Also, the system was designed at a time when we had a considerable investment in processors with no virtualization support in hardware.



A task has properties too, such as its resource requirements and the task’s index within the job. Most task properties are the same across all tasks in a job, but can be overridden – e.g., to provide task-specific command-line flags. Each resource dimension (CPU cores, RAM, disk space,disk access rate, TCP ports,2
etc.) is specified independently at fine granularity; we don’t impose fixed-sized buckets or slots (§5.4). Borg programs are statically linked to reduce dependencies on their runtime environment, and structured
as packages of binaries and data files, whose installation is orchestrated by Borg.



Users operate on jobs by issuing remote procedure calls (RPCs) to Borg, most commonly from a command-line tool, other Borg jobs, or our monitoring systems (§2.6). Most job descriptions are written in the declarative configuration language BCL. This is a variant of GCL [12], which generates
protobuf files [67], extended with some Borg-specific keywords. GCL provides lambda functions to allow calculations, and these are used by applications to adjust their configurations to their environment; tens of thousands of BCL files are over 1 k lines long, and we have accumulated tens of millions of lines of BCL. Borg job configurations have similarities to Aurora configuration files [6]. 



Figure 2 illustrates the states that jobs and tasks go through during their lifetime.





A user can change the properties of some or all of the tasks in a running job by pushing a new job configuration to Borg, and then instructing Borg to update the tasks to the new specification. This acts as a lightweight, non-atomic transaction that can easily be undone until it is closed (committed). Updates are generally done in a rolling fashion, and a limit can be imposed on the number of task disruptions (reschedules or preemptions) an update causes; any changes that would cause more disruptions are skipped. Some task updates (e.g., pushing a new binary) will always require the task to be restarted; some (e.g., increasing resource requirements or changing constraints) might make the task no longer fit on the machine, and cause it to be stopped and rescheduled; and some (e.g., changing priority) can always be done without restarting or moving the task. Tasks can ask to be notified via a Unix SIGTERM signal
before they are preempted by a SIGKILL, so they have time to clean up, save state, finish any currently-executing requests, and decline new ones. The actual notice may be less if the preemptor sets a delay bound. In practice, a notice is delivered about 80% of the time.



### 2.4 Allocs

A Borg alloc (short for allocation) is a reserved set of resources on a machine in which one or more tasks can be run; the resources remain assigned whether or not they are used. Allocs can be used to set resources aside for future tasks, to retain resources between stopping a task and starting it again, and to gather tasks from different jobs onto the same machine – e.g., a web server instance and an associated logsaver task that copies the server’s URL logs from the local disk to a distributed file system. The resources of an alloc are treated in a similar way to the resources of a machine; multiple tasks running inside one share its resources. If an alloc must be relocated to another machine, its tasks are rescheduled with it.



An alloc set is like a job: it is a group of allocs that reserve resources on multiple machines. Once an alloc set has been created, one or more jobs can be submitted to run in it. For brevity, we will generally use “task” to refer to an alloc or a top-level task (one outside an alloc) and “job” to refer to a job or alloc set.

### 2.5 Priority, quota, and admission control

What happens when more work shows up than can be accommodated? Our solutions for this are priority and quota.

Every job has a priority, a small positive integer. A highpriority task can obtain resources at the expense of a lowerpriority one, even if that involves preempting (killing) the latter. Borg defines non-overlapping priority bands for different uses, including (in decreasing-priority order): monitoring, production, batch, and best effort (also known as testing or free). For this paper, prod jobs are the ones in the monitoring and production bands.



Although a preempted task will often be rescheduled elsewhere in the cell, preemption cascades could occur if a high-priority task bumped out a slightly lower-priority one, which bumped out another slightly-lower priority task, and so on. To eliminate most of this, we disallow tasks in the production priority band to preempt one another. Finegrained priorities are still useful in other circumstances –e.g., MapReduce master tasks run at a slightly higher priority than the workers they control, to improve their reliability.



Priority expresses relative importance for jobs that are running or waiting to run in a cell. Quota is used to decide which jobs to admit for scheduling. Quota is expressed as a vector of resource quantities (CPU, RAM, disk, etc.) at a given priority, for a period of time (typically months). The quantities specify the maximum amount of resources that a user’s job requests can ask for at a time (e.g., “20 TiB of RAM at prod priority from now until the end of July in cell xx”). Quota-checking is part of admission control, not scheduling: jobs with insufficient quota are immediately rejected upon submission.



Higher-priority quota costs more than quota at lowerpriority. Production-priority quota is limited to the actual resources available in the cell, so that a user who submits a production-priority job that fits in their quota can expect it to run, modulo fragmentation and constraints. Even though we encourage users to purchase no more quota than they need, many users overbuy because it insulates them against future shortages when their application’s user base grows. We respond to this by over-selling quota at lower-priority levels: every user has infinite quota at priority zero, although this is frequently hard to exercise because resources are oversubscribed. A low-priority job may be admitted but remain pending (unscheduled) due to insufficient resources.



Quota allocation is handled outside of Borg, and is intimately tied to our physical capacity planning, whose results are reflected in the price and availability of quota in different datacenters. User jobs are admitted only if they have sufficient quota at the required priority. The use of quota reduces the need for policies like Dominant Resource Fairness (DRF) [29, 35, 36, 66].



Borg has a capability system that gives special privileges to some users; for example, allowing administrators to delete or modify any job in the cell, or allowing a user to access restricted kernel features or Borg behaviors such as disabling resource estimation (§5.5) on their jobs.

### 2.6 Naming and monitoring

It’s not enough to create and place tasks: a service’s clients and other systems need to be able to find them, even after they are relocated to a new machine. To enable this, Borg creates a stable “Borg name service” (BNS) name for each task that includes the cell name, job name, and task number. Borg writes the task’s hostname and port into a consistent,highly-available file in Chubby [14] with this name, which is used by our RPC system to find the task endpoint. The BNS name also forms the basis of the task’s DNS name,
so the fiftieth task in job jfoo owned by user ubar in cell cc would be reachable via 50.jfoo.ubar.cc.borg.google.com. Borg also writes job size and task health information into Chubby whenever it changes, so load balancers can see where to route requests to.

Almost every task run under Borg contains a built-in HTTP server that publishes information about the health of the task and thousands of performance metrics (e.g., RPC latencies). Borg monitors the health-check URL and restarts tasks that do not respond promptly or return an HTTP error code. Other data is tracked by monitoring tools for dashboards and alerts on service level objective (SLO) violations.

A service called Sigma provides a web-based user interface (UI) through which a user can examine the state of all their jobs, a particular cell, or drill down to individual jobs and tasks to examine their resource behavior, detailed logs, execution history, and eventual fate. Our applications generate voluminous logs; these are automatically rotated to avoid running out of disk space, and preserved for a while after the
task’s exit to assist with debugging. If a job is not running Borg provides a “why pending?” annotation, together with guidance on how to modify the job’s resource requests to better fit the cell. We publish guidelines for “conforming” resource shapes that are likely to schedule easily.

Borg records all job submissions and task events, as well as detailed per-task resource usage information in Infrastore, a scalable read-only data store with an interactive SQL-like interface via Dremel [61]. This data is used for usage-based charging, debugging job and system failures, and long-term capacity planning. It also provided the data for the Google cluster workload trace [80].

All of these features help users to understand and debug the behavior of Borg and their jobs, and help our SREs manage a few tens of thousands of machines per person.

## 3. Borg architecture

A Borg cell consists of a set of machines, a logically centralized controller called the Borgmaster, and an agent process called the Borglet that runs on each machine in a cell (see Figure 1). All components of Borg are written in C++.

### 3.1 Borgmaster

Each cell’s Borgmaster consists of two processes: the main Borgmaster process and a separate scheduler (§3.2). The main Borgmaster process handles client RPCs that either mutate state (e.g., create job) or provide read-only access to data (e.g., lookup job). It also manages state machines for all of the objects in the system (machines, tasks, allocs,etc.), communicates with the Borglets, and offers a web UI as a backup to Sigma.

The Borgmaster is logically a single process but is actuallyreplicated five times. Each replica maintains an inmemory copy of most of the state of the cell, and this state is also recorded in a highly-available, distributed, Paxos-based store [55] on the replicas’ local disks. A single elected master per cell serves both as the Paxos leader and the state mutator, handling all operations that change the cell’s state, such as submitting a job or terminating a task on a machine. A master is elected (using Paxos) when the cell is

