# [workqueu深入理解1](https://github.com/gavin-Angry-Birds/gavin-angry-birds.github.io/issues/2)

### 历史

Linux 的 `workqueue` 子系统最初是为了处理中断的 **底半部**（bottom-half）任务而设计的，随着内核版本的更新，`workqueue` 机制经历了以下演变：

1. **内核 v2.5.41**：首次引入 `workqueue`
   在这个版本中，`workqueue` 作为一种机制被首次引入，目的是将一些长时间运行的任务从中断上下文中移到一个单独的内核线程中处理。这样可以避免在中断上下文中执行时间较长的操作，确保中断处理的高效性和响应性。

2. **内核 v2.6.20**：增加了 **delayed work API**
   在这个版本中，`workqueue` 机制被扩展，引入了 **delayed work**（延迟工作）。这意味着可以将工作项排队并延迟执行，而不是立即执行。这对于处理某些定时任务或需要稍后处理的任务非常有用。

3. **内核 v2.6.36**：引入 **CMWQ**（Concurrency Managed WorkQueue）
   在这个版本中，Linux 内核对 `workqueue` 进行了重大改进，推出了 **CMWQ**（并发管理工作队列）。在 CMWQ 中，`workqueue` 不再每个队列单独创建线程，而是多个队列共享一个 **worker pool**（工作线程池）。这样做的目的是减少线程资源的浪费，提高系统的性能。CMWQ 使得内核可以动态管理 worker 的数量和并发度。

4. **内核 v3.8**：引入 **worker pool**（工作池）
   在这个版本中，Linux 内核引入了 **worker pool** 的概念。每个 CPU 对应一个工作池（worker pool），而不是为每个工作队列创建单独的线程池。这样做使得系统能够更灵活地管理多个工作队列，尤其是在多核 CPU 系统中，通过共享 worker pool 来提升效率。

5. **内核 v3.9**：移除了 **cwq**，统一为 **pool workqueue**
   在 v3.9 版本中，`cwq`（CPU workqueue）被废弃，所有的工作队列都改为使用 **pool workqueue**。这个变化让内核能够更加统一和灵活地管理所有工作队列，进一步优化了性能。

### 图示展示

* **初期工作队列架构**：
  在最初，`workqueue` 是一个简单的结构，每个 CPU 上都有一个独立的工作队列和对应的内核线程（`keventd thread`）。每个工作队列处理的任务都是 CPU 本地的，并且任务在这个线程中被处理。

<img width="613" height="410" alt="Image" src="https://github.com/user-attachments/assets/d4efebb1-0689-43d7-80ff-b33b8cea778a" />

* **CMWQ 方式引入后**：
  随着 CMWQ 的引入，多个工作队列可以共享一个工作池（worker pool），并且可以动态管理 worker 线程的数量。这种方式通过减少线程的创建和销毁，优化了系统性能和资源使用。

<img width="859" height="572" alt="Image" src="https://github.com/user-attachments/assets/82403c64-0cd7-4ba0-8872-fef8a72d7eea" />

* **工作池（worker pool）添加后**：
  在 v3.8 及之后的版本中，Linux 内核使用了 **worker pool** 来管理线程池，支持多核 CPU 系统中的负载均衡和高效调度。

<img width="1244" height="616" alt="Image" src="https://github.com/user-attachments/assets/4890a1ba-5423-4049-b51e-c825640135d2" />

* **最近的实现方式**：
  在最近的内核版本中，`workqueue` 的实现方式已经非常成熟，系统通过 **pool workqueue** 统一管理不同的工作队列和工作池，实现了高效的线程调度和资源管理。

<img width="904" height="635" alt="Image" src="https://github.com/user-attachments/assets/8e9ede9e-5069-4c30-a76c-35acb72da5e7" />

> kernel v3.9~4.11:cw- pool_workqueue로변경 -> kernel v3.9~4.11: changed to cw-pool_workqueue
bound.cpux2( (highpri))고정운용 -> bound.cpux2 (highpri) fixed operation 
동시처리제한된워크둘 -> Simultaneous processing limited to two workloads
unbound속성(nice, cpu mask)별 dynamic운영 for bound -> unbound attributes (nice, cpu mask) dynamic operation for bound
bound:cpux2[ (highpri)고정'운용 -> bound: cpux2 [highpri] fixed operation
unbound:속성(nice, cpu mask))별 dynamic운영 per-node -> unbound: dynamic operation per node based on attributes (nice, cpu mask)