## 算法概括介绍
CClockList是一种基于双向链表的Clock缓存替换算法。Clock算法是一种近似LRU算法，它通过一个环形链表和引用位（reference bit）来实现。当一个数据项被访问时，它的引用位被设置为1。当需要淘汰数据时，一个“时钟指针”会从当前位置开始扫描链表，遇到引用位为0的数据项就将其淘汰，并前进一位；如果遇到引用位为1的数据项，就将其引用位清零，然后前进到下一个数据项，直到找到一个引用位为0的数据项。
## `CClockListHook` 结构体
- `using Time = uint32_t;`: 使用32位无符号整数来存储更新时间。
- `using CompressedPtr = typename T::CompressedPtr;`: 使用压缩指针，这是Cachelib中常见的优化，用于在64位系统上节省内存。
- `setNext`, `getNext`: 用于设置和获取链表中的下一个节点。
- `setUpdateTime`, `getUpdateTime`: 管理节点的更新时间戳。
- `getState`, `markState`, `clearState`: 这是该算法的核心。`getState`获取当前状态，`markState`使用原子操作（`__atomic_compare_exchange_n`）来安全地更新状态，`clearState`清零状态。这里的`state_`变量是核心，它对应于CLOCK算法中的引用位以及其他扩展状态。
**`CClockList` 类模板**: 这是整个算法的实现类。它是一个模板，可以用于任何包含 `CClockListHook` 的节点类型。
## 创新点
1. `CClockListHook`中的`state_`字段以及`CC_UNMARK`, `CC_MOVABLE`, `CC_EVICTABLE`等宏定义表明，一个节点有多种状态。
2. 压缩指针：CompressedPtr。在64位系统上，指针通常占用8字节，而如果内存地址空间较小（如小于4GB），则可以使用4字节的偏移量来表示指针，从而节省一半的内存。
3. **驱逐策略**: `getEvictionCandidate0`到`getEvictionCandidate4`这些函数表明该算法有多种驱逐策略或阶段。

## 具体实现
```C++
#define CC_UNMARK   0
#define CC_MOVABLE    1
#define CC_EVICTABLE  2
```
CC_UNMARK是缓存项的默认或初始状态。当一个新项被添加到缓存中，或者**它的应用位被清除后**，处于这个状态。在这个状态下，缓存项还没有被“时钟指针”检查过，也没有被标记为可移动或可驱逐。
CC_MOVABLE表示缓存项被最近访问过。当“时钟指针”扫描到处于CC_UNMARK状态的项，并且发现它已经被访问（`isAccessed(*curr)` 为真）时，会将其状态更新为 `CC_MOVABLE`。这相当于给了它一个“第二次机会”。
`CC_EVICTABLE`表示缓存项**可以被驱逐**。当“时钟指针”扫描到处于 `CC_UNMARK` 状态的项，并且发现它没有被访问（`isAccessed(*curr)` 为假）时，会将其状态更新为 `CC_EVICTABLE`。

`CClockListHook` 结构体的主要作用是作为 CLOCK 算法中每个**缓存项的元数据（metadata）**。
- CompressedPtr next_：一个指向链表中下一个节点的压缩指针
- Time updateTime_：存储缓存项**最后一次被更新到链表头部的时间**。
- uint32_t state_：存储缓存项的**当前状态**。

**重点**：
这里的CClockListHook类和CClockList类都是模板类，使用时需要根据要求存储的缓存项数据类型定义具体的类。
CClockListHook类定义了链表的节点信息，使用压缩指针节省空间。
CClockList类定义了链表头指针，尾指针，节点数量和请求计数器等。这里的指针需要指向真实内存地址。

## 迭代器类Iterator
构造函数：`T* p, Direction d, const CClockList<T, HookPtr>& CClockList`
三个参数：模版参数T代表类型的指针，遍历方向d，对一个 `CClockList` 实例的**常量引用**。

// copyable and movable    可以**拷贝和移动** 

拷贝构造函数
`Iterator(const Iterator&) = default;`
拷贝赋值运算符
`Iterator& operator=(const Iterator&) = default;`
以上**创建或赋值一个对象的副本**，分配独立的资源

移动构造函数
`Iterator(Iterator&&) noexcept = default;`
移动赋值运算符
`Iterator& operator=(Iterator&&) noexcept = default;`
以上不会分配新资源，直接将旧对象的资源所有权转移给新对象。

operator++()：前置自增，将迭代器前进到链表的下一个节点
在迭代器中实现了常见的移动到下一个节点，指针访问，判断相等，不等，为空。
get获取值，reset节点置空，resetToBegin链表置空。

## `CClockList` 模板类
是一个单向链表，通过`CClockListHook` 将缓存项连接起来。
- 模版参数：
	- `T`: 表示缓存项的类型。
	- `HookPtr`: 一个指向 `CClockListHook` 成员的指针，用于将 `CClockList` 实例与具体的缓存项类型 `T` 关联起来。
- 成员变量：
	- `compressor_`: 用于压缩和解压缩指针。
	- `mtx_`: 一个**分布式互斥锁** 
	- `head_` 和 `tail_`: 原子指针，分别指向链表的头部和尾部
	- `size_` 和 `countRequest_`: 原子计数器，分别记录链表的大小和请求次数。

## 驱逐候选项筛选函数0-4
### getEvictionCandidate0():
```C++
template <typename T, CClockListHook<T> T::*HookPtr>
T* CClockList<T, HookPtr>::getEvictionCandidate0() noexcept {
  T* curr;

  while (true) {
    curr = head_.load();
    if (curr == nullptr)
      return nullptr;
    while(!head_.compare_exchange_weak(curr, getNext(*curr))){}
    // If the node is valid and has been accessed, move it to the tail
    if(isValid(*curr) && isAccessed(*curr)){
      unmarkAccessed(*curr);
      linkAtTail(*curr, *curr); 
    }
    // Else, unlink and return
    else{
      // printf("Evict0, size=%lu\n", size_.load());
      size_--;
      return curr;
    }
  }
}
```

compare_exchange_weak 用于执行**原子性的比较和交换操作**。
接收三个主要参数：
1. **`expected`**：一个指针或引用，指向你期望共享变量当前拥有的值。
2. **`desired`**：你希望将共享变量更新成的新值。
3. **`memory_order`**：可选参数，用于指定内存同步顺序（例如 `__ATOMIC_ACQ_REL`）。

该函数会执行以下操作：
1. 读取共享变量的当前值。
2. 比较：将读取到的值与提供的expected值进行比较。
3. 如果相等：
	1. 函数将共享变量的值**原子性地**更新为desired
	2. 返回**true**，表示操作成功。
4. 如果不相等：
	1. 函数将共享变量的当前值更新为你提供的expected变量中。
	2. 返回false，表示操作失败。

**设计思路**：
1. **`head_.load()`**：原子性地读取链表头部的指针。
	- 每次启动getEvictionCandidate0时，首先获取遍历链表的头指针赋值给curr，**`head_` 指针充当了时钟指针**。 
2.  `head_.compare_exchange_weak(curr, getNext(*curr))`：这是实现无锁并发的关键。
	- 它尝试原子性地更新头部指针。如果当前头部指针 `head_` 的值等于 `curr`（我们期望的值），它就会将 `head_` 更新为 `getNext(*curr)`（下一个节点），并返回 `true`。
	- 如果失败，它会将 `head_` 的最新值赋给 `curr` 并返回 `false`，然后循环会继续，使用新的 `curr` 值再次尝试。
	- 这确保了多个线程在同时修改头部指针时，只有一个会成功，而其他线程会重新尝试，从而保证了操作的原子性和线程安全。
	- 一旦一个线程成功地跳出 `while` 循环，它就拿到了**处理 `curr` 节点的独占权**，这个 `curr` 节点是**在 `compare_exchange_weak` 成功时，旧的 `head_` 指向的节点**。
3. 驱逐判断：对于被扫描到的每个节点，算法会检查其状态：
	1. **`isAccessed(*curr)`**：如果该节点被访问过（访问标志为 `true`），它不应被立即驱逐。这就像给了它一个“第二次机会”。
		1. **`unmarkAccessed(*curr)`**：为了这个第二次机会，算法会清除该节点的访问标志。
		2. **`linkAtTail(*curr, *curr)`**：更进一步，该函数将这个节点移动到链表的尾部。这个操作有两个目的：一是将它从时钟指针的前方移走，避免它在下一次循环中再次被扫描；二是延长它的生命周期，让它在缓存中停留更长时间，因为驱逐是从头部开始的。
	2. 如果节点没有被访问过（`isAccessed(*curr)` 为 `false`），它就被认为是**最老的、且最近没有被使用过的**节点。
		1. `size_--`：找到驱逐候选项后，原子性地减小缓存大小。
		2. `return curr`：函数返回该节点，将其作为驱逐的候选项。
4. **关键点**：每次调用getEvictionCandidate0函数面对两种情况：
	1. 如果当前节点被访问过（访问标志为 `true`），给与第二次机会，但是`linkAtTail(*curr, *curr)`将这个节点移动到了链表尾部。
	2. 如果节点没有被访问过（`isAccessed(*curr)` 为 `false`），直接这个节点**淘汰**。
	3. 因此，head_其实始终指向**链表的头结点**，同时是**每次替换查询的头指针**。

### getEvictionCandidate1():
```C++
template <typename T, CClockListHook<T> T::*HookPtr>
T* CClockList<T, HookPtr>::getEvictionCandidate1() noexcept {
  if (size_.load() == 0)
    return nullptr;

  LockHolder l(*mtx_);

  T* oldHead = head_.load();
  T* prev = nullptr;
  T* curr = oldHead;

  while (true) {
    if(isValid(*curr) && isAccessed(*curr)){
      unmarkAccessed(*curr);
      prev = curr;
      curr = getNext(*curr);
    }
    else{
      head_ = getNext(*curr);
      if (l.owns_lock()) {
        l.unlock();
      }
      if(prev){
        setNext(*prev, nullptr);
        linkAtTail(*oldHead, *prev);        
      }
      // printf("Evict1, size=%d\n", size_.load());
      setNext(*curr, nullptr);
      size_--;
      return curr;
    }
  }
}
```

设计思路：
1. `LockHolder l(*mtx_);`使用**显式锁**来实现一个更传统的时钟算法驱逐策略。
2. **遍历与筛选**：
	1. 函数从 `head_` 开始遍历链表。
	2. **`if(isValid(*curr) && isAccessed(*curr))`**: 如果当前节点有效且已被访问，它不应该被驱逐。算法会清除它的访问标志 (`unmarkAccessed(*curr)`)，然后继续扫描下一个节点 (`curr = getNext(*curr)`)。`prev` 指针用于记录当前节点的前一个节点，这在后续的批量移动操作中非常重要。
3. **驱逐与批量移动**：
	1. 在加锁的情况下，直接更新 `head_` 指针是安全的。新的头部是当前驱逐节点的下一个节点。
	2. **释放锁**：`if (l.owns_lock()) { l.unlock(); }`。一旦找到驱逐候选项并更新了 `head_`，就立即释放锁。这有助于减少锁的持有时间，提高并发性。
	3. **批量移动**：
		1. `if(prev)`: 如果 `prev` 不为空，说明在找到驱逐候选项之前，我们扫描过至少一个不应被驱逐的节点。
		2. `linkAtTail(*oldHead, *prev);`：将从 `oldHead` 到 `prev` 的**所有节点**（即扫描过但未被驱逐的节点）作为一个块，整体移动到链表的尾部。

`getEvictionCandidate1` **这种加锁的设计更易于理解和实现，它牺牲了一定的并发性来换取更强的**逻辑原子性，即整个驱逐和批量移动操作在锁的保护下作为一个整体完成。

### getEvictionCandidate2()：
```C++
template <typename T, CClockListHook<T> T::*HookPtr>
T* CClockList<T, HookPtr>::getEvictionCandidate2() noexcept {
  T* prevHead, *prev, *curr = nullptr;

  while (true) {
    if(curr == nullptr){
      curr = prevHead = head_.load();
      prev = nullptr;
    }

    if(isValid(*curr) && isAccessed(*curr)){
      prev = curr;
      curr = getNext(*curr); 
      continue;     
    }

    bool updateHead = head_.compare_exchange_weak(prevHead, getNext(*curr));
    if(updateHead){
      if(prev){
        unmarkAccessedBatch(prevHead, prev); // Set the state of the nodes between prevHead and curr to unAccessed.
        linkAtTail(*prevHead, *prev);
      }
      size_--;
      return curr;
    }

    curr = prevHead;
    prev = nullptr;
  }
}
```

设计思路：
1. 这个函数旨在优化 `getEvictionCandidate0` 的效率。`getEvictionCandidate0` 每次只能处理一个节点并移动 `head_` 指针，而 `getEvictionCandidate2` 尝试扫描一批节点，并用一个原子操作将 `head_` 一次性移动到这批节点的末尾。
2. **分段式无锁扫描**：
	1. **`if(curr == nullptr)`**: 这段代码是循环的起始点。它会从当前 `head_` 开始扫描，将 `head_` 的值赋给 `curr` 和 `prevHead`。
	2. **`if(isValid(*curr) && isAccessed(*curr))`**: 如果当前节点已被访问，它会继续向后扫描，同时更新 `prev` 和 `curr` 指针。
	3. **`continue`**: 只要节点被访问过，循环就会继续前进，**不触碰 `head_`**，从而在不加锁的情况下扫描一批连续的、已访问的节点。
3. **原子性更新与批量处理**：
	1. **`bool updateHead = head_.compare_exchange_weak(prevHead, getNext(*curr));`**: 这是关键步骤。当 `curr` 指向一个**未访问**的节点时，函数尝试原子性地更新 `head_`。
		1. `prevHead` 是这批扫描的起始节点。
		2. `getNext(*curr)` 是驱逐候选项的下一个节点。
		3. 如果 `head_` 仍然是 `prevHead`（即没有其他线程在扫描期间修改过 `head_`），这个原子操作就会成功，将 `head_` 一次性移动到新的位置。
	2. **`if(updateHead)`**: 原子操作成功后，当前线程独占了从 `prevHead` 到 `curr` 这一段链表的处理权。
		1. **`unmarkAccessedBatch(prevHead, prev)`**: 将这批已扫描的节点（从 `prevHead` 到 `prev`）的访问标志批量清除。
		2. **`linkAtTail(*prevHead, *prev)`**: 将这批节点作为一个整体，批量移动到链表尾部。
4. **处理竞争与重试**：
	1. 如果 `head_.compare_exchange_weak` 失败，说明在扫描过程中，有另一个线程捷足先登，修改了 `head_`。
	2. **`curr = prevHead; prev = nullptr;`**: 此时，当前线程会放弃已经扫描过的这批节点，重新从 `head_` 的新位置开始扫描，进行下一次尝试。

### getEvictionCandidate3()：
```C++
template <typename T, CClockListHook<T> T::*HookPtr>
T* CClockList<T, HookPtr>::getEvictionCandidate3() noexcept {
  T *curr = head_.load();
  T *prev = nullptr;
  std::list<T*> historyHeads;
  historyHeads.push_back(curr);

  while (true) {
    // printf("Evict2.0, curr:%x\n", curr);

    if(curr == nullptr || getNext(*curr) == nullptr){
      curr = head_.load();
      historyHeads.clear();
      historyHeads.push_back(curr);
      prev = nullptr;
      continue;
    }

    // If the next node is nullptr, wait other thread inserts nodes at the tail.

    // printf("Evict2.1, curr:%x, next:%x\n", curr, getNext(*curr));

	// 获取当前curr状态，只有 CC_UNMARK 状态的节点才允许被当前线程进行状态转换
    uint32_t currState = getState(*curr);
    uint32_t desiredState = CC_UNMARK;

    bool markable = (currState == CC_UNMARK);

    if(markable){
      if(isValid(*curr) && isAccessed(*curr)){
        desiredState = CC_MOVABLE;
      }
      else{
        desiredState = CC_EVICTABLE;
      }
      // markState是一个原子比较并交换操作
      markable = markState(*curr, &currState, desiredState);
    }

    // If the current head is not in history heads, it means the head has been changed by other threads.
    // markable 检查当前线程是否成功将curr节点标记为 CC_MOVABLE 或 CC_EVICTABLE
    // find 检查当前全局head_指针是否还在该线程自己的 historyHeads 历史记录中
	    // head_.load() 会获取全局的最新 head_ 指针。
	    // 如果最新的 head_ 不在 historyHeads 记录中，说明该线程扫描的时间段中，另一个线程已经完成了自己的驱逐操作并原子性地修改了全局head_指针。这意味着**当前线程的扫描已经失效**，即竞争处理失败。
    if(markable && std::find(historyHeads.begin(), historyHeads.end(), head_.load()) == historyHeads.end()){
    // 撤销curr的标记，重置所有状态，从新的head_位置开始扫描。
      clearState(*curr);
      // printf("Evict2.2, curr%x\n", curr);
      curr = head_.load();
      historyHeads.clear();
      historyHeads.push_back(curr);
      prev = nullptr;
      continue;
    }

	// !markable: 这个条件为真，说明另一个线程已经修改了该节点的状态
	// currState == CC_EVICTABLE 意味着这个节点被其他线程标记为“待驱逐”
    if(!markable && currState == CC_EVICTABLE){
	    // 跳过当前节点，限制historyHeads 列表大小避免无限增长
      historyHeads.push_back(getNext(*curr));
      if(historyHeads.size() > 64){
        historyHeads.pop_front();
      }
    }

    // printf("Evict2.3, desire:%d, success?:%d, curr:%x\n", desiredState, markable, curr);

    // If current node is movable, decrease the accessed time.
    if(markable && (desiredState == CC_MOVABLE)){
      // printf("Evict2.4, curr:%x\n", curr);
      unmarkAccessed(*curr);     
    }

    // If current node is evictable, try to evict it.
    if(markable && (desiredState == CC_EVICTABLE)){
      // Fetch the previous head from the historyHeads.
      T* prevHead = historyHeads.back();

      // Blocking until the head_ is the same as the previous head.
      // printf("Evict2.5, curr:%x\n", curr);
      // 忙等待同步：不断地检查全局的 head_ 指针，直到它与自己记录的 prevHead 相等。
      while(head_.load() != prevHead){}

      // Set the head_ to the next node of the current node.
      // printf("prevHead:%x\n", prevHead);

      head_.store(getNext(*curr));

      clearStates(prevHead, curr);

      if(prevHead != curr){
        linkAtTail(*prevHead, *prev);
      }
      // setNext(*curr, nullptr);
      // printf("newHead:%x, state:%d\n", head_.load(), getState(*head_.load()));
      // printf("Evict2.6, curr:%x\n", curr);
      size_--;
      return curr;
    }
    prev = curr;
    curr = getNext(*curr); 
  }
}
```

设计思路：
1. 多阶段状态标记：与简单的时钟算法只用一个“访问位”不同，这个函数为每个节点定义了三种状态，并利用原子操作进行转换。
	1. `markState` 函数是实现这一逻辑的核心。它使用**原子比较并交换（compare-and-exchange）操作**，安全地将节点从 `CC_UNMARK` 转换为 `CC_MOVABLE` 或 `CC_EVICTABLE`。这相当于一个线程**原子性地“声明”了对一个节点的处理权**，避免了加锁。
2. 流水线式乐观遍历：
	1. **状态转换**：一个线程扫描链表，如果遇到一个未标记的节点，它会尝试原子性地将其标记为 `CC_MOVABLE`（如果被访问过）或 `CC_EVICTABLE`（如果未被访问）。
	2. **乐观前进**：这个线程不会立即对已标记的节点进行操作，而是继续向前扫描 (`prev = curr; curr = getNext(*curr)`)。这种方式形成了“流水线”，允许多个线程同时进行标记操作。
	3. **历史记录**：`historyHeads` 列表用于记录该线程处理过的链表片段的起始节点。这在处理并发冲突时至关重要。
3. 处理竞争与驱逐：该算法最复杂的部分在于，如何在没有全局锁的情况下，安全地驱逐节点。
	1. **驱逐触发**：当一个线程成功地将节点标记为 `CC_EVICTABLE` 时，这表明该节点是它扫描到的“最旧”的未访问节点。
	2. **头部所有权检查**：线程会检查当前的全局 `head_` 指针是否与它在 `historyHeads` 中记录的 `prevHead` 相符。
	3. **忙等待**：`while(head_.load() != prevHead){}` 这是一个**忙等待循环**。它迫使当前线程暂停，直到其他线程处理完毕并将 `head_` 指针移动到它所期望的位置。这确保了线程不会去驱逐一个正在被其他线程处理的节点。
	4. **原子性驱逐**：等待结束后，线程实际上就**独占**了它所处理的那段链表。此时，它执行驱逐操作：
		1. `head_.store(getNext(*curr))`：原子性地将 `head_` 指针移动到被驱逐节点之后，将其从链表中“移除”。
		2. `clearStates(prevHead, curr)`：清除已处理节点的状态。
		3. `linkAtTail(*prevHead, *prev)`：将那段被标记为 `CC_MOVABLE` 的节点作为一个整体，批量移动到链表尾部，给予它们“第二次机会”。

`getEvictionCandidate3` 的设计，本质上是一个复杂的**无锁流水线**。它允许多个线程并发地进行扫描和标记，并通过一个同步点（忙等待）来确保在执行关键的驱逐操作时，只有一个线程能修改链表头部，从而在高性能和线程安全之间取得了平衡。

### getEvictionCandidate4()：
```C++
template <typename T, CClockListHook<T> T::*HookPtr>
T* CClockList<T, HookPtr>::getEvictionCandidate4() noexcept {
  if (size_.load() == 0)
    return nullptr;

  T *oldHead, *newHead, *newTail = nullptr;
  T *curr, *next, *ret = nullptr;

  size_t spanSize = 4;

  while(true){
    do{
      newHead = oldHead = head_.load();
      for(int i = 0; i < spanSize; i++){
        if(getNext(*newHead) == nullptr){
          if(oldHead == head_.load()){
            // printf("Evict3.0, oldHead:%x\n", oldHead);
            i--;
          }
          else{
            // printf("Evict3.1, oldHead:%x\n", oldHead);
            break;
          }
        }
        else
        {
          newTail = newHead;
          newHead = getNext(*newHead);
          // printf("Evict3.3, oldHead:%x, newHead:%x\n", oldHead, newHead);
        }
      }
    }while(!head_.compare_exchange_weak(oldHead, newHead));

    // printf("Evict3.3, newHead:%x\n", newHead);

    curr = oldHead;
    while(curr != newHead){
      next = getNext(*curr);
      if(isValid(*curr) && isAccessed(*curr)){
        unmarkAccessed(*curr);
        linkAtTail(*curr, *curr); 
        printf("Evict3.4, move, curr:%x\n", curr);
        curr = next;
      }
      else{
        if(curr != newTail)
          linkAtTail(*next, *newTail);
        size_--;
        return curr;
      }
    }
  }
}
```

设计思路：
1. 批量原子性地移动头部指针：
	1. **预定片段**：`do...while` 循环是关键的无锁部分。一个线程首先获取当前的 `head_` 作为 `oldHead`，然后遍历 `spanSize` 次，找到片段的末尾节点，并将其下一个节点作为 `newHead`。
	2. **原子性抢占**：`head_.compare_exchange_weak(oldHead, newHead)` 会尝试原子性地将全局 `head_` 指针从 `oldHead` 更新为 `newHead`。如果成功，该线程就“抢占”了从 `oldHead` 到 `newTail` 的整个链表片段。
	3. **竞争与重试**：如果 `compare_exchange_weak` 失败，说明在线程遍历这段时间里，有其他线程捷足先登，修改了 `head_`。此时，循环会重新开始，线程会再次从最新的 `head_` 开始预定一个新的片段。
2. 独占性地处理片段内的节点：
	1. 驱逐逻辑：
		1. **`if(isValid(*curr) && isAccessed(*curr))`**: 如果节点被访问过，清除其访问标志，并将其**单个**移动到链表尾部 (`linkAtTail(*curr, *curr)`)。
		2. **`else`**: 如果节点未被访问过，它就是驱逐候选项。该节点会被返回。
	2. 处理剩余节点：
		1. `if(curr != newTail) linkAtTail(*next, *newTail);`：如果驱逐候选项 `curr` 不是片段的最后一个节点 (`newTail`)，那么 `curr` 之后的节点（从 `next` 到 `newTail`）也会被作为一个整体，移动到链表尾部。


该函数的核心思想是：在**无锁**的情况下，预先划定一个包含 `spanSize` 个节点的**链表片段**，然后通过一个原子操作将链表头部指针 `head_` **一次性跳过整个片段**。这样，一个线程就可以独占性地处理这个片段，而其他线程可以立即开始处理下一个片段，从而实现了**高度并行化**的驱逐过程。

这个函数 `getEvictionCandidate4` 的设计思路是一种**无锁的、批量时钟（Clock）算法变体**，它旨在通过一次性处理多个节点来提高性能。它结合了 `getEvictionCandidate0` 的无锁特性和 `getEvictionCandidate2` 的批量处理思想。