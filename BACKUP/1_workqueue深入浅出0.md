# [workqueue深入浅出0](https://github.com/gavin-Angry-Birds/gavin-angry-birds.github.io/issues/1)

### `workqueue` 介绍与分析

`workqueue` 是在处理硬件中断时用于实现底半部（bottom-half）任务处理的机制。设备的中断上半部（top-half）处理完成后，下半部的任务会被交给 `workqueue` 进行处理，这样就能避免在中断处理过程中阻塞。

#### `workqueue` 的优先级

`workqueue` 的优先级是通过系统任务的 `nice` 值来调节的，`nice` 值决定了任务的调度优先级。

* **高优先级队列（`highpri`）** 使用 `nice` 值为 -20，类似于 `softirq`，这是所有任务中优先级最高的。
* 与 `softirq` 和 `tasklet` 等底半部机制不同，`workqueue` 允许在任务执行时进入睡眠状态，这使得它能处理更复杂的任务。

接下来的图片展示了通过 `workqueue` 处理底半部任务的流程：

<img width="1599" height="223" alt="Image" src="https://github.com/user-attachments/assets/b7f5d7ba-b5db-4968-bdd1-766a9ae7f17e" />

> **中断处理** -> 将任务交给 `workqueue` -> `worker` 线程处理工作。

---

### `workqueue` 的组件与特点

#### 系统预设的 `workqueue`

系统会根据不同的任务类型，创建多个不同的工作队列。在内核早期，只有 `bound` 和 `unbound` 两种工作队列，随着内核版本的更新，逐渐增加到现在的 7 个系统工作队列。以下是一些重要的系统 `workqueue`：

1. **`system_wq`**
   用于多 CPU、多线程的环境，处理快速且不需要大量时间的任务。

2. **`system_highpri_wq`**
   使用最强的优先级（`nice = -20`），用来处理优先级最高的任务，类似于 `softirq`。

3. **`system_long_wq`**
   与 `system_wq` 类似，但用于处理稍微需要更长时间的任务。

4. **`system_unbound_wq`**
   不绑定到特定 CPU，适用于那些不依赖于某个 CPU 的任务。每次只能处理一个任务，但可以通过 `max_active` 来限制并发任务数。

5. **`system_freezable`**
   类似于 `system_wq`，但支持挂起（suspend）时的冻结操作。任务在挂起时会暂停，恢复后继续执行。

6. **`system_power_efficient_wq`**
   用于节能目的，即使牺牲一定的性能，也能完成任务。

7. **`system_freezable_power_efficient_wq`**
   类似于 `system_power_efficient_wq`，但是在挂起时会冻结任务并在恢复后继续执行。

#### 工作队列与工作池（`pwq`）

* **工作池（`pwq`）**：用于管理延迟任务的队列。工作池会管理这些任务，确保它们在合适的时间被分配给空闲的工作线程。
* **CPU 绑定的工作队列（`bound`）**：工作池的数量等于系统中的 CPU 数量。每个 CPU 可能会拥有多个与其绑定的工作池。
* **非 CPU 绑定的工作队列（`unbound`）**：在 UMA 系统中只有一个工作池，而在 NUMA 系统中，则会根据节点数创建对应数量的工作池。

#### 工作线程（`worker`）

* **工作线程的管理**：工作线程会分为空闲（`idle`）和繁忙（`busy`）两类。空闲线程会被唤醒处理新的任务，繁忙线程会被加入到工作池中。
* **工作线程的生命周期**：在每个工作池中，初始会创建一个空闲的工作线程，并且会根据需要动态创建更多线程。工作线程的销毁周期大约是 5 分钟，每次销毁时会保持至少 2 个空闲线程。

#### 工作项（`work`）

* **底半部处理的任务**：工作项就是需要 `workqueue` 执行的任务，每个工作项会绑定一个处理函数。当工作线程处理工作项时，这个函数会被调用。

* **延迟工作项（`delayed work`）**：延迟工作项在被处理之前会先启动一个定时器，定时器到期后才会被放入工作队列。

* **并发执行**：即使有多个工作项被放入工作队列，也不会出现同一个工作项被同时处理的情况（即使 `max_active` 的值不为 1）。多个不同的工作项会根据 CPU 数量和 `max_active` 的限制进行并发处理。

下图展示了与 workqueue 相关的各个组件之间的关系

<img width="1038" height="732" alt="Image" src="https://github.com/user-attachments/assets/0e12989a-975a-42a9-be88-09f9b67c53fd" />



### 总结

`workqueue` 主要用于在 Linux 内核中处理设备中断的下半部任务。通过将较长时间执行或可能阻塞的任务从中断上下文中移出，`workqueue` 能够使系统更加高效和稳定。内核提供了多种 `workqueue` 类型以及配置选项，以适应不同的硬件和任务要求，同时可以动态调整工作池和工作线程的数量，以提高系统的灵活性和响应性。
