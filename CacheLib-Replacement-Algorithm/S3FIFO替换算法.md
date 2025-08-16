## S3FIFO算法设计思路
S3FIFO（Simple-3-Queues-FIFO）是一种高效的缓存淘汰算法，旨在解决传统FIFO（先进先出）和LRU（最近最少使用）算法的局限性。它的核心思想是通过**三个简单队列**来模拟一个更复杂的、对访问模式更敏感的淘汰策略，同时保持低开销。
- **P-FIFO（Probationary FIFO）**：这个队列用于存放新进入缓存的项。可以理解为新来的“候选人”。如果一个项在 P-FIFO 中再次被访问，就说明它有可能是个“热”数据，值得被保留。
- **M-FIFO（Main FIFO）**：这个队列用于存放被 P-FIFO 验证过的“热”数据。M-FIFO 的容量通常比 P-FIFO 大得多。数据在 M-FIFO 中遵循 FIFO 规则，直到被淘汰。
- **Ghost Queue（历史队列）**：它不存储实际的数据项，而是存储被 P-FIFO 淘汰的数据项的哈希值。它的作用是**过滤掉那些短期内被频繁访问但又迅速冷掉的数据**（"polluters"），防止它们再次进入 P-FIFO 污染缓存。当一个数据项被淘汰时，它的哈希值会被添加到这个队列。当有新的数据项请求进入缓存时，会先检查其哈希值是否在历史队列中。如果在，就直接将其放入 M-FIFO，因为它可能之前就是“热”数据。

## 工作过程
### 核心数据结构：
1. `pfifo_` (Probationary FIFO): `std::unique_ptr<ADList>`，用**原子双向链表** (`AtomicDList`) 实现。用于存放新进入的缓存项。
2. `mfifo_` (Main FIFO): `std::unique_ptr<ADList>`，同样用**原子双向链表** (`AtomicDList`) 实现。用于存放经过验证的热点缓存项。
3. `hist_` (History Queue): `AtomicFIFOHashTable`，一个固定大小的哈希表，用于存储被淘汰项的哈希值。

### 操作流程：
1. 添加（add）：当一个新项被添加到缓存中：
	1. 首先检查`hist_`（历史队列）。
	2. 如果**命中** `hist_`，说明它之前被淘汰过，但可能很快又被访问了，因此直接将其添加到 `mfifo_` 的头部。
	3. 如果**未命中** `hist_`，说明这是一个新项，将其添加到 `pfifo_` 的头部。
2. 淘汰 (`getEvictionCandidate`):
	1. 淘汰逻辑首先检查 **P-FIFO** 和 **M-FIFO** 的大小比例。如果 `pfifo_` 的大小超过总大小的一定比例（由 `pRatio_` 控制），说明 P-FIFO 队列太长，需要优先淘汰。
	2. **从 P-FIFO 淘汰**：
		1. 从 `pfifo_` 的尾部取出一个项。
		2. 如果该项被**访问过**（通过 `isAccessed` 检查，通常是通过一个标志位实现），说明它在 `pfifo_` 存活期间被访问了，是“热”数据。此时，将其移动到 `mfifo_` 的头部，并清除其“已访问”标志。
		3. 如果该项**未被访问**，说明它是一个“冷”数据。将其哈希值插入到 `hist_` 历史队列，然后返回该项作为淘汰候选。
	3. **从 M-FIFO 淘汰**：
		1. 如果 `pfifo_` 的大小比例没有超过阈值，则从 `mfifo_` 的尾部取出一个项。
		2. 如果该项被**访问过**，说明它是一个“热”数据，将其移动到 `mfifo_` 的头部，并清除其“已访问”标志。
		3. 如果该项**未被访问**，直接返回该项作为淘汰候选。

### 重点函数分析
#### evictPFifo()
这个函数是 `prepareEvictionCandidates()` 的一个子函数，专门负责从P-FIFO中淘汰单个项。
工作流程：
1. **从 P-FIFO 尾部移除**:
	1. `curr = pfifo_->removeTail();` 这行代码尝试从 `pfifo_` 的尾部移除一个缓存项。
2. **判断项是否被访问过** 
3. **如果项被访问过（晋升）**: 
	1. `pfifo_->unmarkAccessed(*curr);` : 清除访问标志。
	2. `unmarkProbationary(*curr);` 和 `markMain(*curr);` : 更新项的状态标志，将其标记为“主”（Main）队列中的成员，不再是“临时”（Probationary）状态。
	3. `mfifo_->linkAtHead(*curr);` : 将该项**晋升**到 **M-FIFO** 的头部。
4. **如果项未被访问（真正淘汰）**: 
	1. `hist_.insert(hashNode(*curr));` : 将该项的哈希值插入到**历史队列**（`hist_`）中。
	2. `if (!evictCandidateQueue_.write(curr))` : 尝试将该项写入**淘汰候选队列**。这个队列用于多线程环境下，将淘汰项提供给主线程。
	3. `pfifo_->linkAtHead(*curr);` : 如果写入淘汰队列失败（通常是因为队列已满），该项会被**重新链接**回 P-FIFO 的头部。

#### evictMFifo()
这个函数是 `prepareEvictionCandidates()` 的一个子函数，专门负责从M-FIFO中淘汰单个项。

这个函数的作用是从 **M-FIFO（主队列）** 的队尾取出一个缓存项，并根据其是否被访问过，决定是将其**重新晋升**到队首还是**标记为淘汰候选**。
工作流程：
1. 从 M-FIFO 队尾移除
2. 判断是否被访问
	1. 如果被访问过（重新晋升）
		1. 清除访问标志
		2. 移到队首
	2. 如果未被访问（标记淘汰）
		1. `evictCandidateQueue_.write(curr)`: 尝试将该项写入**淘汰候选队列**。
		2. `pfifo_->linkAtHead(*curr);` : 如果写入淘汰队列失败（通常是因为队列已满），该项会被**重新链接**回 P-FIFO 的头部。

#### prepareEvictionCandidates()
实现思路是：**通过批量处理来优化性能，并根据当前缓存状态动态调整淘汰策略。** 
1. 动态淘汰策略 (Dynamic Eviction Strategy)
	1. `if (pfifo_->size() > (double)(pfifo_->size() + mfifo_->size()) * pRatio_)`：如果 P-FIFO 的大小超过了总大小的某个小比例，说明有大量的新数据涌入缓存，且这些数据可能没有被再次访问。为了防止 P-FIFO 膨胀并污染缓存，算法会优先**清空** P-FIFO。
	2. 如果 P-FIFO 的大小比例正常，说明缓存的“健康度”良好，此时应该专注于淘汰 M-FIFO 中那些最久未被使用的“冷”数据。
2. 批量处理 (Batch Processing)
	1. `for (int i = 0; i < nCandidateToPrepare(); i++)`：这个循环通过多次调用 `evictPFifo()` 或 `evictMFifo()`，一次性处理多个淘汰逻辑。

这种批量处理的思路是为了**提高效率**。它减少了函数调用的开销，并且在多线程环境下，可以让后台线程一次性生成足够的淘汰候选，从而避免频繁地与主线程进行通信。

`prepareEvictionCandidates()` 是整个**异步预取机制**的关键组成部分。
- 这个函数通常在后台线程中运行，作为**生产者**，负责生成淘汰候选项。
- 它将这些候选项放入一个 **MPMC（多生产者多消费者）队列** (`evictCandidateQueue_`)。
- 主线程作为**消费者**，可以直接从队列中快速获取一个已经准备好的淘汰项，而无需等待耗时的淘汰逻辑计算。
这种设计将复杂的淘汰逻辑从主线程的**关键路径**中分离出来，显著降低了延迟，使得 S3FIFO 能够在高并发环境中保持高效。

#### getEvictionCandidate0()
```C++
template <typename T, AtomicDListHook<T> T::*HookPtr>
T* S3FIFOList<T, HookPtr>::getEvictionCandidate0() noexcept {
  // 缓存为空的处理
  size_t listSize = pfifo_->size() + mfifo_->size();
  if (listSize == 0 && evictCandidateQueue_.size() == 0) {
    return nullptr;
  }

  T* curr = nullptr;
  // 惰性初始化 hist_ 和 evictCandidateQueue_ 
  if (!hist_.initialized()) {
  // 第一层检查是非加锁的，可以快速通过。
    LockHolder l(*mtx_);
    if (!hist_.initialized()) {
      hist_.setFIFOSize(listSize / 2);
      hist_.initHashtable();
// #define ENABLE_SCALABILITY
#ifdef ENABLE_SCALABILITY
      if (evThread_.get() == nullptr) {
        evThread_ = std::make_unique<std::thread>(&S3FIFOList::threadFunc, this);
      }
#endif
    }
  }

  // 主动填充预取队列 evictCandidateQueue_，确保队列始终有足够的“存货”。
  size_t sz = evictCandidateQueue_.sizeGuess();
  if (sz < nMaxEvictionCandidates_ / (2 + sz % 8)) {
    prepareEvictionCandidates();
  }

  int nTries = 0;
  // 主线程进入一个循环，持续尝试从队列中读取。
  while (!evictCandidateQueue_.read(curr)) {
    // 防止主线程在空循环中进行自旋等待
    if ((nTries++) % 100 == 0) {
      prepareEvictionCandidates();
    }
  }

  return curr;
}
```

`getEvictionCandidate0()` 是一个高性能、多线程的 S3FIFO 缓存淘汰候选项获取函数。它的设计核心是基于**生产者-消费者模型**，通过将复杂的淘汰逻辑转移到后台线程来**降低主线程的延迟**。
工作流程：
1. 惰性初始化：函数在首次被调用且缓存不为空时，才会初始化**历史队列**（`hist_`）和**后台淘汰线程**（`evThread_`）。通过使用分布式互斥锁（`DistributedMutex`），它确保了初始化过程的线程安全。
2. 异步淘汰逻辑（生产者-消费者模型）
3. 主动队列补给：
	1. 该函数通过主动管理 `evictCandidateQueue_` 来防止队列变空。

#### getEvictionCandidate()        S3FIFO淘汰逻辑的单线程版本
```C++
template <typename T, AtomicDListHook<T> T::*HookPtr>
T* S3FIFOList<T, HookPtr>::getEvictionCandidate() noexcept {

  size_t listSize = pfifo_->size() + mfifo_->size();
  if (listSize == 0) {
    return nullptr;
  }

  // 惰性初始化 hist_ 
  T* curr = nullptr;
  if (!hist_.initialized()) {
    LockHolder l(*mtx_);
    if (!hist_.initialized()) {
      hist_.setFIFOSize(listSize / 2);
      hist_.initHashtable();
    }
  }

  while (true) {
    if (pfifo_->size() > (double)(pfifo_->size() + mfifo_->size()) * pRatio_) {
      // evict from probationary FIFO
      // 优先淘汰 P-FIFO
      curr = pfifo_->removeTail();
      if (curr == nullptr) {
        if (pfifo_->size() != 0) {
          printf("pfifo_->size() = %zu\n", pfifo_->size());
          abort();
        }
        continue;
      }
      if (pfifo_->isAccessed(*curr)) {
        pfifo_->unmarkAccessed(*curr);
        XDCHECK(isProbationary(*curr));
        unmarkProbationary(*curr);
        markMain(*curr);
        mfifo_->linkAtHead(*curr);
      } else {
        hist_.insert(hashNode(*curr));

        return curr;
      }
    } else {    // 淘汰 M-FIFO
      curr = mfifo_->removeTail();
      if (curr == nullptr) {
        continue;
      }
      if (mfifo_->isAccessed(*curr)) {
        mfifo_->unmarkAccessed(*curr);
        mfifo_->linkAtHead(*curr);
      } else {
        return curr;
      }
    }
  }
}
```

性能不够原因分析：
1. 阻塞与高延迟：
	1. 当主线程需要一个淘汰候选项时，它必须进入 `while (true)` 循环，自己执行整个淘汰逻辑，包括遍历队列、检查访问标志、移动节点，直到找到一个可以返回的项。
2. 线程安全性与锁竞争：
	1. 使用互斥锁 `mtx_` 来初始化 `hist_` ，存在锁争用。
3. 缺乏并行处理能力：
	1. 完全依赖一个线程来执行所有工作，无法利用现代多核 CPU 的并行处理能力。
4. 无法应对突发负载
	1. 单线程版本的性能与当前淘汰操作的复杂性直接相关。如果需要多次循环才能找到一个合适的淘汰项，性能就会急剧下降。
	2. 多线程版本通过**批量预取**的方式解决了这个问题。后台线程会持续地将淘汰候选项放入队列，即使突然出现多个需要淘汰的请求，队列中也通常有足够的“备用”项可以立即使用，从而平滑了性能波动，提高了系统的健壮性。


## S3FIFO工作中的细节
当一个新请求访问缓存项时，S3FIFO 算法**不会**在队列中寻找是否命中。这个任务由一个更高层的、外部的**哈希表**来完成。

现在的`S3FIFOList` 代码是一个**链表管理器**，它只负责**维护项的淘汰顺序**，而不是查找项本身。

在 Cachelib 框架中，真正的查找操作由一个全局的哈希表来完成。当一个请求到来时，其键会直接在哈希表中查询。这个过程通常是 O(1) 的，非常高效。
- **缓存命中**：如果在哈希表中找到了对应的缓存项，那么该项就已经被定位了。此时，S3FIFO 算法的职责是**更新**该项的状态，而不是寻找它。
- **缓存未命中**：如果哈希表中没有找到该项，那么缓存未命中。应用程序需要从底层存储（如磁盘或网络）加载数据，然后将新项**插入**到 S3FIFO 队列中。

### S3FIFO 算法在命中后的行为
当哈希表成功命中一个缓存项时，S3FIFO 算法会根据该项当前所在的队列，执行不同的操作：
1. 如果该项在 **P-FIFO**（临时队列）中：
	1. **操作**：该项不会在 P-FIFO 中移动位置，但其**访问标志会被设置**。
	2. **原因**：S3FIFO将 P-FIFO 视为一个“试用期”队列。如果一个项在试用期内被访问，它只会被标记，表示它“表现良好”。只有当它被推到 P-FIFO 的队尾、即将被淘汰时，算法才会根据这个标志来决定是否将其**晋升**到 M-FIFO。这种延迟处理机制避免了频繁的队列移动，减少了不必要的开销。
2. 如果该项在 **M-FIFO**（主队列）中：
	1. **操作**：该项会被从当前位置**移动到 M-FIFO 的队首**。
3. 如果该项在 **历史队列** 中：
	1. **操作**：这种情况**不可能发生**。历史队列只存储被淘汰项的**哈希值**，而不存储实际的缓存项。这意味着如果一个键在历史队列中，它对应的缓存项已经不在内存中了。
	2. **原因**：这是一个缓存未命中的情况。应用程序需要重新加载数据，然后调用 `S3FIFOList::add()` 将其作为新项插入缓存。值得注意的是，`add()` 函数会检查这个历史队列。如果键的哈希值在历史队列中，它会被直接放入 M-FIFO，因为它之前就是热数据，只是暂时被淘汰了。这有效防止了“污染者”再次进入缓存。

### 缓存项的状态信息
```C++
// Bit MM_BIT_0 is used to record if the item is hot.
  void markProbationary(T& node) noexcept {
    node.template setFlag<RefFlags::kMMFlag0>();
  }

  void unmarkProbationary(T& node) noexcept {
    node.template unSetFlag<RefFlags::kMMFlag0>();
  }

  bool isProbationary(const T& node) const noexcept {
    return node.template isFlagSet<RefFlags::kMMFlag0>();
  }

  // Bit MM_BIT_2 is used to record if the item is cold.
  void markMain(T& node) noexcept {
    node.template setFlag<RefFlags::kMMFlag2>();
  }

  void unmarkMain(T& node) noexcept {
    node.template unSetFlag<RefFlags::kMMFlag2>();
  }

  bool isMain(const T& node) const noexcept {
    return node.template isFlagSet<RefFlags::kMMFlag2>();
  }
```

每一个缓存项（`T` 类型的 `node`）都有一些**内部标志位**。当一个缓存项被添加到 `pfifo_` 队列时，它会通过 `markProbationary()` 将其 `kMMFlag0` 标志位设置为 `true`。同样，当一个缓存项被移动到 `mfifo_` 队列时，它会通过 `markMain()` 将其 `kMMFlag2` 标志位设置为 `true`。

外部代码只需通过简单的位检查（例如调用 `isProbationary()` 或 `isMain()`）就能**以 O(1) 的时间复杂度**迅速确定该项属于 P-FIFO 还是 M-FIFO。


## 多生产者多消费者(MPMC)队列
多生产者多消费者（MPMC）队列是一种并发数据结构，允许多个线程同时向队列中添加数据（作为生产者），也允许多个线程同时从队列中取出数据（作为消费者），且不会产生冲突或数据不一致问题。

### 使用场景
MPMC 队列通常用于需要将任务或数据从一个或多个处理阶段安全、高效地传递到另一个或多个处理阶段的场景。其使用模式通常是：
1. **任务/事件流水线**：一个或多个线程负责生成任务（例如，接收网络请求、读取文件），并将任务对象放入 MPMC 队列。
2. **工作者线程池**：一个或多个消费者线程（通常是工作者线程池）持续从队列中取出任务并执行。
这种模式的好处在于，生产者和消费者可以以不同的速度运行，而无需相互等待。队列作为一个缓冲区，平滑了生产和消费速度的波动。

### 实际应用
MPMC 队列在各种需要高并发、解耦任务的算法和系统中得到广泛应用：
1. **缓存替换算法**：如上述的**S3FIFO**，它利用 MPMC 队列将“选择淘汰项”的复杂计算从主线程中分离。一个或多个后台线程作为生产者，预先计算并准备好淘汰候选，然后将其放入队列。主线程作为消费者，需要淘汰时可以直接从队列中快速获取，避免了计算开销。
2. **并行算法**：在数据处理、机器学习训练等场景中，一个线程可能负责将数据切分并放入队列，而多个工作线程则从队列中并行处理数据块。这在 MapReduce 等编程模型中尤为常见。
3. **日志系统**：多个应用线程可以作为生产者，异步地将日志事件写入一个 MPMC 队列。一个或少数几个日志工作线程则作为消费者，从队列中批量读取并写入磁盘，减少了对应用线程的 I/O 阻塞。
4. **无锁编程**：在高性能计算领域，MPMC 队列常常采用无锁（lock-free）技术实现，以避免传统锁带来的上下文切换和性能开销，从而最大化吞吐量。

### 优点
1. **高吞吐量**：它允许生产者和消费者完全并行运行。如果你的系统有足够的 CPU 核心，可以同时进行数据生产和消费，从而显著提高整体吞吐量。
2. **解耦**：生产者和消费者之间通过队列进行通信，它们不需要知道对方的实现细节，也不用互相等待。这使得系统设计更简洁、更模块化，便于扩展和维护。
3. **负载均衡**：当有多个消费者时，任务会自然地在它们之间进行分配。如果一个消费者处理得慢，其他消费者可以分担其工作，实现了简单的负载均衡。
4. **高效率**：许多 MPMC 队列都采用了无锁或最小化锁的设计，避免了锁竞争带来的性能损耗，这对于对延迟敏感的高并发应用至关重要。