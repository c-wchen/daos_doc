- [Overview](https://github.com/pmodels/argobots/wiki/Introduction-to-Argobots#overview)
- Execution Model
  - [User-level Threads and Tasklets](https://github.com/pmodels/argobots/wiki/Introduction-to-Argobots#user-level-threads-and-tasklets)
  - [Work Unit Migration](https://github.com/pmodels/argobots/wiki/Introduction-to-Argobots#work-unit-migration)
  - [Stackable Scheduler with Pluggable Strategies](https://github.com/pmodels/argobots/wiki/Introduction-to-Argobots#stackable-scheduler-with-pluggable-strategies)
- Memory Model
  - [Consistency Domains](https://github.com/pmodels/argobots/wiki/Introduction-to-Argobots#consistency-domains)
- [Organization of this Document](https://github.com/pmodels/argobots/wiki/Introduction-to-Argobots#organization-of-this-document)

# Overview

当今最强大的 HPC 系统的并发性呈爆炸式增长。由于未来的百亿亿次级系统可能由数亿个运算单元组成，未来的应用程序可能包含数十亿个线程或任务以利用底层硬件提供的并发性。然而，实现十亿路并行需要高度动态的计算和数据调度。传统的、低性能的重量级线程和消息传递模型无法支持大规模并行，因为它们经常优化组件的隔离和分层以支持一维，例如计算或通信。因此，需要提供一种轻量级执行模型，将低延迟线程和任务调度与优化的数据移动功能相结合。 

Argobots 是一个低级基础设施，支持用于大规模并发的轻量级线程和任务模型。它将直接利用硬件中的最低级别构造 和操作系统：轻量级通知机制、数据移动引擎、内存映射和数据放置策略。 它由一个**执行模型**和一个**内存模型**组成。

# Execution Model

Argobots 支持两个级别的并行性：执行流 (ES) 和工作单元 (WU)。 ES 是包含一个或多个 WU 的顺序指令流。它绑定硬件资源并保证进度。 WU 是轻量级执行单元，例如用户级线程 (ULT) 或 tasklet。它们与函数调用相关联，每个 WU 都将执行直至完成。一个WU可以关联一个ES，ES根据调度策略运行所有关联的WU。基本上，单个 ES 中 WU 不可以并行执行。但是，不同 ES 中的 WU 可以并行执行。 Argobots 设想了两种抽象。第一个抽象支持轻量级并发执行，例如 ULT 或 tasklet，它们可以动态有效地适应来自应用程序和硬件资源的需求。 ULT 和 tasklet 必须根据功率、弹性、内存位置和功能进行有效调度。

二次抽象 支持低延迟、轻量级、消息驱动的激活。 调度程序的指导设计原则是在托管硬件资源上协作、非抢占式地激活可调度计算单元（ULT 或 tasklet）。 本地化调度策略（例如当前运行时系统中使用的调度策略）虽然对短时间执行有效，但不了解全局策略和优先级。 适应性和动态系统行为必须通过可以随时间变化或针对特定算法或数据结构定制的调度策略来处理。 “插入”专门的调度策略让 OS/R 处理该机制并利用轻量级通知功能，同时将策略留给系统软件堆栈的更高级别。

## User-level Threads and Tasklets

用户级线程 (ULT) 和 tasklet 为应用程序提供了细粒度并行性的抽象。每个ULT或tasklet都是一个独立的执行单元，由runtime系统调度。它们在细微的方面有所不同，这使得它们中的每一个都更适合某些编程主题。例如，ULT 有自己的持久堆栈区域，而 tasklet 借用其宿主 ES 调度程序的堆栈。我们期望更高级别的编程模型根据这些基本调度单元分解它们的计算。 

ULT 非常适合根据持久性上下文表达并行性，其控制流根据数据流暂停和恢复。一个常见的例子是过度分解的应用程序，它使用阻塞接收（或期货）来等待远程数据。与传统的 OS 线程不同，ULT 不打算被抢占。他们合作将控制权交给调度程序，例如，当他们等待远程数据或只是希望让其他人工作时 进步。 Argobots 中的 Tasklet 是一个不可分割的工作单元，仅依赖于其输入数据，通常在完成时提供输出数据。 任务的应用级示例包括分子动力学中的力评估对象，当两组原子可用时执行，或 LU 分解中的尾随更新，当两个矩阵块可用时执行矩阵-矩阵 (GEMM) 计算 可用。 

Tasklet 不会显式地让出控制权，而是在将控制权返回给调用它们的调度程序之前运行至完成。 ULT 和 tasklet 可以根据它们自己的策略独立调度，包括远程数据的到达、内存的可用性或数据依赖性。 

ULT 和 tasklet 都可以保持与一个或多个数据对象的亲和力，调度程序将在决策制定中考虑这些对象。 一旦准备好执行，ULT 和 tasklet 都由具有公共工作池的调度程序管理。

## Work Unit Migration

Argobots 支持在不同位置（即 ES 或调度程序池）之间迁移工作单元（ULT 或 tasklet）。基本上，所有由用户创建的 ULT 都可以迁移，除非它们被终止。但是，tasklet 只有在还没有开始执行的情况下才能被迁移。迁移操作可以是阻塞的或非阻塞的，具体取决于请求。并且，如果设置了回调函数，它将在迁移发生时被调用。 迁移操作分为两类：以工作单元为中心的迁移和以位置为中心的迁移。

这两个操作都可以用作迁移的推送和拉取接口。 以工作单元为中心的迁移是将一个工作单元迁移到另一个 ES 或另一个池。根据谁请求迁移，它可以是推送或拉取操作。

例如工作单元W0请求将工作单元W1迁移到与W0不同的ES或pool中，这就是push操作。它将工作单元 W1 推到不同的 地点。 但是，如果将W1迁移到调用工作单元W0所在的ES或pool中，则为pull操作。 以工作单元为中心的迁移还包括使工作单元成为孤立的接口，即它使工作单元与任何 ES 和任何池都没有关联。 

以位置为中心的迁移是将与特定 ES 或池关联的一定数量的工作单元迁移到不同的 ES 或池。 这种方法侧重于迁移发生的位置，而不是要迁移的工作单元。 此类迁移例程指定源位置、目标位置和要迁移的工作单元数，但它们不指定要迁移哪些工作单元。 如果由其关联的 ES 或池与目标不同的工作单元请求，则可以将以位置为中心的迁移视为推送操作。 否则，即，如果目标与 ES 或与调用工作单元关联的池相同，则它的行为类似于 拉操作。

## Stackable Scheduler with Pluggable Strategies

整个调度框架以具有多个可插入调度策略的可堆叠或嵌套调度器的概念为中心。插入自定义策略会修改该调度程序实例管理的所有任务的调度规则。可插拔性还期望这些决策所需的输入在 OS/R 软件堆栈中的单个点可用。除了可插拔策略之外，OS/R 调度程序将允许特定于编程语言或应用程序组件的嵌套（堆叠）调度程序引擎。例如，系统可以为应用程序中的每个模块接受一个调度程序；并且基于依赖关系或相关模块优先级，OS/R 调度器可以调用堆栈调度器之一。现在，它可以在托管硬件上激活其任务，并在完成后交出控制权。类似的堆栈调度器可能是多种编程语言在单个 OS/R 调度器的上下文中交互的结果。基于一个框架 任务依赖性分析可能会选择在满足依赖性后安排任务。 这将被表示为一个堆栈式调度器，它处理 DAG 依赖性分析和调度任务，或者在它们“准备好”被激活时简单地将它们传播到 OS/R 调度器。

# Memory Model

当前的通信库没有很好地与调度程序或内存管理器集成。这三者必须协同工作，以支持跨节点的优化数据移动，并提供构建特定编程环境的更高层所需的功能。 Argobots 提供了一个支持最终一致性的集成内存模型。

## Consistency Domains

当前的硬件将一致性和一致性合二为一，但这在 NUMA 机器或非 DRAM（如内存）上可能代价高昂。 Argobots 通过明确地将内存空间划分为不同的域来解耦一致性和连贯性。 Argobots 中的一致性分为三个级别： 1. Consistency Domains (CDs)； 2. 非相干加载/存储域 (NCLSD)； 3. 在 NCLSD 之外。 

一致性域是数据最终变得一致的内存区域，这意味着如果没有对给定数据项进行新更新，最终对该项目的所有访问都将返回最后更新的值。同时，可以通过内存屏障强制执行即时一致性。在 NCLSD 中，使用加载/存储访问数据，但不提供硬件一致性。在 NCLSD 之外，显式 Put/Get/Messaging 模型用于移动数据。

# Organization of this Document

本文档的其余部分以术语和约定开头，解释了本文档中使用的 Argobots 术语和约定。然后描述了 Argobots 组件及其 API。