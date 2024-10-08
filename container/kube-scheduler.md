# 7.7.3 调度器及扩展设计

调度器（kube-scheduler）的主要职责是为新创建的 Pod 找到最合适的节点。如果集群只有几十个节点，调度并不困难，但当节点规模达到几千甚至更大时，情况就复杂得多。

Pod 的创建/更新和节点资源无时无刻不在发生变化。如果每次调度都需要进行数千次远程请求以获取相关信息，不仅会耗时过长、可能导致调度失败，同时调度器频繁的网络请求也会使其成为集群的性能瓶颈。

:::tip <a/>
为了充分利用硬件资源，通常会将各种类型(CPU 密集、IO 密集、批量处理、低延迟作业)的 workloads 运行在同一台机器上，这种方式减少了硬件上的投入，但也使调度问题更加复杂。

随着集群规模的增大，需要调度的任务的规模也线性增大，由于调度器的工作负载与集群大小大致成比例，调度器有成为可伸缩性瓶颈的风险。

:::right
—— from Omega 论文
:::

## 1. 调度双循环架构

Omega 的论文中提出了一种基于共享状态（图 7-1 中的 Scheduler Cache）的双循环调度机制，用来解决大规模集群的调度效率问题，这种调度机制不仅应用在 Google 的 Omega 系统中，也被 Kubernetes 继承下来。

Kubernetes 默认调度器（kube-scheduler）双循环架构如下所示。

:::center
  ![](../assets/kube-scheduler.png)<br/>
  图 7-37 kube-scheduler 双循环架构设计
:::

第一个控制循环称之为 Informer Path，它主要目的是启动一系列 Informer 监听（Watch）Etcd 中 Pod、Node、Service 等与调度相关的 API 对象的变化。譬如一个待调度 Pod 被创建后，调度器就会通过 Pod Informer 的 Handler 将这个待调度 Pod 添加进调度队列。

Kubernetes 的调度器还要负责对调度器缓存（即 Scheduler Cache）进行更新，缓存的目的主要是对调度部分进行性能优化，将集群信息 cache 化，以便提升 Predicate 和 Priority 调度算法的执行效率。

第二个控制循环是调度器负责 Pod 调度的主循环，被称之为 Scheduling Path。

Scheduling Path 主要逻辑是不断地从调度队列里出队一个 Pod。然后调用 Predicates 算法对所有的 Node 进行“过滤”。“过滤”得到的一组可以运行这个 Pod 的 Node 列表。当然，Predicates 算法需要的 Node 信息，也都是 Scheduler Cache 里直接拿到的，这是调度器保证算法执行效率的主要手段之一。

接下来，调度器就会再调用 Priorities 算法为上述列表里的 Node 打分，得分最高的 Node 就会作为这次调度的结果。

调度算法执行完成后，调度器就需要将 Pod 对象的 nodeName 字段的值，修改为上述 Node 的名字，这个过程在 Kubernetes 里面被称作 Bind。为了不在关键调度路径里远程访问 API Server，Kubernetes 默认调度器在 Bind 阶段只会更新 Scheduler Cache 里的 Pod 和 Node 的信息。这种基于“乐观”假设的 API 对象更新方式，在 Kubernetes 里被称作 Assume。Assume 之后，调度器才会创建一个 Goroutine 异步地向 API Server 发起更新 Pod 的请求，完成真正 Bind 操作。


## 2. 调度器可扩展设计

“Pod 是原子的调度单位”这句话的含义是 kube-scheduler 以 Pod 为调度单元进行依次调度，并不考虑 Pod 之间的关联关系。

但是很多数据**计算类的离线作业具有组合调度的特点，要求所有的子任务都能够成功创建后，整个作业才能正常运行**，即所谓的 All_or_Nothing。

例如：JobA 需要 4 个 Pod 同时启动，才算正常运行。kube-scheduler 依次调度 3 个 Pod 并创建成功，到第 4 个 Pod 时，集群资源不足，则 JobA 的 3 个 Pod 处于空等的状态。但是它们已经占用了部分资源，如果第 4 个 Pod 不能及时启动的话，整个 JobA 无法成功运行，更糟糕的集群其他的资源刚好被 JobB 的 3 个 Pod 所占用，同时在等待 JobB 的第 4 个 Pod 创建，此时整个集群就出现了死锁。

**解决以上问题的思想是将调度单元从 Pod 修改为 PodGroup，以组的形式进行调度，实现“Gang Scheduling”**。

实现 Gang Scheduling 的第一步，便是要干预 Pod 的调度逻辑。

Kubernetes 从 v1.15 版本起，为 kube-scheduler 设计了可插拔的扩展机制 —— Scheduling Framework。

:::center
  ![](../assets/scheduling-framework-extensions.png)<br/>
   图 7-38 Pod 的调度上下文以及调度框架公开的扩展点
:::

有了 Scheduling Framework，在保持调度“核心”简单且可维护的同时，用户可以编写自己的调度插件注册到 Scheduling Framework 的扩展点来实现自己想要的调度逻辑。

譬如你可以扩展调度队列的实现，控制每个调度的时机，在 Predicates 阶段选择满足某一组 Pod 资源的节点，实现多个 Pod 被作为一个整体调度。
