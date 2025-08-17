Mobius是一种为Facebook的Cachelib框架设计的缓存替换算法，它通过使用两个队列和原子操作来提供一种**线程安全**且**高效**的缓存淘汰机制。该算法的核心思想是模仿**时钟(Clock)算法**，同时利用无锁(lock-free)的数据结构和原子操作来实现高并发下的性能优化。

## 工作原理
Mobius算法的核心是**两个独立的队列**，可以称为“活跃队列”和“休眠队列”，由一个`whichQ_`布尔变量来指示当前哪个是活跃队列。当一个缓存项被访问时，它会被标记为已访问。当需要淘汰缓存项时，Mobius从活跃队列的头部开始遍历。
- **访问和标记**: 当一个缓存项被访问（即**命中**）时，它不会立即移动到队列的尾部。相反，它仅通过`markAccessed`函数设置一个特殊的标志（`kMMFlag1`）。
- **淘汰和移动**: 当系统需要一个可淘汰的缓存项时，`getEvictionCandidate0()`或`getEvictionCandidate1()`函数被调用。这些函数会从活跃队列的头部开始检查缓存项。
	- 如果一个缓存项**没有**被标记为已访问，它就被认为是“冷”的，可以立即被淘汰，并从缓存中移除。
	- 如果一个缓存项**被**标记为已访问，它被认为是“热”的，不应被淘汰。此时，该缓存项的已访问标记被清除，并被移动到**休眠队列**的尾部。
- **队列切换**: 当活跃队列的头部指针到达队列的末尾或只剩一个元素时，算法会通过原子操作**切换**活跃队列和休眠队列。原先的休眠队列变为新的活跃队列，这个过程**类似于时钟算法中指针的“绕一圈”**。

## Mobius模板类的设计分析

### 模板参数
- **`typename T`**: 代表缓存中的节点类型。这个类型必须包含一个公共成员，该成员的类型为 `MobiusHook<T>`。
- **`MobiusHook<T> T::*HookPtr`**: 这是一个指向成员的指针。它告诉 `Mobius` 类在 `T` 类型的对象中，`MobiusHook` 成员变量位于何处。通过这种方式，`Mobius` 可以在不了解 `T` 内部结构的情况下，访问和修改节点的链接信息。

### 成员变量
- **`const PtrCompressor compressor_{};`**: 一个指针压缩器。这个成员用于在压缩后的指针 (`CompressedPtr`) 和实际指针 (`T*`) 之间进行转换。
- **`mutable folly::cacheline_aligned<Mutex> mtx_;`**: 一个**互斥锁**，用于在需要时保护临界区。
- **`std::atomic<T*> head_[2]{nullptr, nullptr};`**: 一个包含两个 `std::atomic<T*>` 变量的数组，分别代表两个队列的**头部**。
- **`std::atomic<T*> tail_[2]{nullptr, nullptr};`**: 同样是一个包含两个 `std::atomic<T*>` 变量的数组，分别代表两个队列的**尾部**。
- **`std::atomic<bool> whichQ_{0};`**: 一个原子布尔变量，用于指示当前哪个队列是“活跃队列”（`head_[0]` 和 `tail_[0]`）和“休眠队列”（`head_[1]` 和 `tail_[1]`）。
- **`std::atomic<size_t> size_{0};`**: 一个原子计数器，用于追踪缓存中节点的总数量。
- **`std::atomic<size_t> countRequest_{0};`**: 一个原子计数器，用于记录请求的数量。

### `MobiusHook` 结构体

`MobiusHook` 是一个嵌套在 `Mobius` 外部的通用结构体，它代表了双向链表中的一个节点。但在 `Mobius` 的实现中，它只使用了 `next_` 指针，而 `prev_` 指针被注释为“未在 Mobius 中使用”，这表明 Mobius 实际上实现的是一个**单向链表**。


## 关键函数及分析

### linkAtTail(head_, tail_, T& beginNode, T& endNode)：
`linkAtTail` 是 `Mobius` 缓存替换算法中的一个**核心私有函数**，其主要作用是**原子且无锁地**将一个或一组节点链接到指定队列的尾部。
```C++
template <typename T, MobiusHook<T> T::*HookPtr>
void Mobius<T, HookPtr>::linkAtTail(std::atomic<T*>& head_, std::atomic<T*>& tail_, T& beginNode, T& endNode) noexcept {
  setNext(endNode, nullptr);
  T* oldTail = tail_.load();

  while (!tail_.compare_exchange_weak(oldTail, &endNode)) {}

  // this is the thread that first makes Tail_ points to the node
  // other threads must follow this, o.w. oldHead will be nullptr
  if(oldTail == nullptr){
    // 队列为空时添加节点同时是尾节点和头结点
    head_ = &beginNode;
  }
  else{
    // 将 beginNode 连接到队列旧的尾节点之后
    setNext(*oldTail, &beginNode);
  }
}
```

当 `beginNode` 和 `endNode` 相同，这意味着正在插入**单个节点**。
插入单个节点的流程：
1. **设置 `next` 指针**: `setNext(endNode, nullptr);` 
	1. 新节点 `endNode` 的 `next` 指针被设置为 `nullptr`。
2. **获取当前尾部**: `T* oldTail = tail_.load();` 
	1. `tail_` 指向链表中**原先的**最后一个节点。`oldTail` 被赋予该节点的地址。
3. **无锁 CAS 循环**: `while (!tail_.compare_exchange_weak(oldTail, &endNode)) {}` 
	1. CAS 操作尝试将 `tail_` 的值从 `oldTail` （即旧尾部节点的地址）更新为 `&endNode` （即**新节点的地址**）。
	2. 假设没有其他线程同时尝试修改尾部，CAS 操作成功，循环立即结束
	3. 此时，`tail_` 已经指向了新节点 `node`。`oldTail` 仍然保存着旧尾部节点的地址。
4. **链接新节点**: `else { setNext(*oldTail, &beginNode); }` 
	1. `setNext(*oldTail, &beginNode)` 会将**旧尾部节点**的 `next` 指针设置为`beginNode`。

### add(T& node)：
```C++
template <typename T, MobiusHook<T> T::*HookPtr>
void Mobius<T, HookPtr>::add(T& node) noexcept {
  bool wQ = whichQ_.load();
  linkAtTail(head_[wQ], tail_[wQ], node, node);
  size_.fetch_add(1);
}
```
该函数用于将新节点添加到缓存中。它会根据`whichQ_`的值，将节点原子地添加到当前活跃队列的尾部，并增加缓存的总大小。

`linkAtTail(head_[wQ], tail_[wQ], node, node);`：这里传入了两次 `node`，分别是 `beginNode` 和 `endNode`。这种设计允许 `linkAtTail` 函数既可以添加单个节点，也可以添加一个节点批次，从而提高了代码的复用性。

### remove(T& node)：
```C++
template <typename T, MobiusHook<T> T::*HookPtr>
void Mobius<T, HookPtr>::remove(T& node) noexcept {
  // 调试断言，确保 remove 时缓存大小大于0
  XDCHECK_GT(size_, 0u);
  if (isValid(node)) {
    unmarkValid(node);
    unmarkAccessed(node);
  }
}
```
该函数负责从缓存中移除一个节点。它通过`unmarkValid`和`unmarkAccessed`清除节点的标记，并减少缓存的总大小`size_`。

`remove` 函数采用了一种**懒惰（Lazy）且无锁**的设计思路，它不是立即将节点从链表中物理移除，而是通过标记节点为无效来实现“逻辑删除”。

### getEvictionCandidate0()：基础淘汰算法
```C++
template <typename T, MobiusHook<T> T::*HookPtr>
T* Mobius<T, HookPtr>::getEvictionCandidate0() noexcept {
  // 确定当前活跃队列
  bool wQ = whichQ_.load();

  auto &activeHead_ = head_[wQ];
  auto &activeTail_ = tail_[wQ];

  auto &dormantHead_ = head_[!wQ];
  auto &dormantTail_ = tail_[!wQ];

  T *curr;

  while(true){
    curr = activeHead_.load();
    // CAS操作尝试将 curr 移动到下一个节点
    do{
      // When the active Q has only 1 elements, switch the active Q.
      // 活跃队列只剩下一个元素，进行队列切换
      if(getNext(*curr) == nullptr && activeHead_.load() == curr){
        whichQ_.compare_exchange_weak(wQ, !wQ);
        return getEvictionCandidate0();
      }
    }while(!activeHead_.compare_exchange_weak(curr, getNext(*curr)));
	// 退出循环时，curr变量的值 还是 被推进的那个旧头部节点的地址
    if(isValid(*curr) && isAccessed(*curr)){
      unmarkAccessed(*curr);
      linkAtTail(dormantHead_, dormantTail_, *curr, *curr);
    }
    else{
      size_.fetch_sub(1);
      return curr;
    }
  }
}
```

#### 分析do-while循环如何工作：
1. 循环开始前：`curr = activeHead_.load();` 
	1. 第一次进入 `do-while` 循环时，`curr` 被赋予了 `activeHead_` 的当前值。
2. **`compare_exchange_weak` 操作**:
	1. `compare_exchange_weak(curr, getNext(*curr))` 尝试将 `activeHead_` 从`curr` 变量当前的值原子地更新为 `getNext(*curr)`。
	2. 如果这个操作成功（返回 `true`），说明 `activeHead_` 的值和 `curr` 当前的值是相等的，然后`activeHead_`成功被更新为`getNext(*curr)`。
	3. 如果这个操作失败（返回 `false`），说明在执行 CAS 时，`activeHead_` 已经被其他线程修改，它的值和 `curr` 的值不再相等。此时，`compare_exchange_weak` 会**原子地将 `activeHead_` 的最新值重新加载到 `curr` 变量中**。
3. 循环退出：
	1. 当 `compare_exchange_weak` 成功时，循环条件 `!true` 为 `false`，循环退出。
	2. 此时，`curr` 的值还是 CAS 操作成功前 `activeHead_` 的值，也就是**在这次原子前进操作中被“跳过”的那个节点**。

getEvictionCandidate0()从活跃队列头部开始遍历：
1. 使用`compare_exchange_weak`原子地将活跃队列的头部向前移动一个节点。
2. 如果被检查的节点是**已访问**的（`isAccessed()`为真），它会清除该标记(`unmarkAccessed()`)并使用`linkAtTail`将其移动到休眠队列的尾部。
3. 如果节点**未被访问**，则返回该节点作为淘汰候选项，并减少总大小`size_`。
4. 当活跃队列为空或只剩一个元素时，它会原子地切换`whichQ_`，并递归调用自身，从新的活跃队列中获取候选项。

### getEvictionCandidate1()：优化淘汰算法
```C++
template <typename T, MobiusHook<T> T::*HookPtr>
T* Mobius<T, HookPtr>::getEvictionCandidate1() noexcept {

  bool wQ = whichQ_.load();

  auto &activeHead_ = head_[wQ];
  auto &activeTail_ = tail_[wQ];

  auto &dormantHead_ = head_[!wQ];
  auto &dormantTail_ = tail_[!wQ];

  T *prevHead, *prev, *curr = nullptr;
  int i;
  while(true){
    // 第一次执行时初始化，或者CAS操作失败
    if(curr == nullptr){
      // curr重新赋值为最新的头指针 activeHead_ 
      curr = prevHead = activeHead_.load();
      prev = nullptr;
      i = 0;
    }
    // Active Q has only no more elements, switch the active Q.
    // 活跃队列只剩一个元素或为空，交换当前队列
    if(curr == nullptr || (getNext(*curr) == nullptr && prevHead == activeHead_.load())){
        if(whichQ_.compare_exchange_weak(wQ, !wQ)){
          // 当前线程刚刚遍历过的所有节点进行 取消标记操作
          unmarkAccessedBatch(prevHead, curr);
        }
        return getEvictionCandidate1();
    }

    if(isValid(*curr) && isAccessed(*curr)){
      // curr是有效且最近访问过的缓存项，继续向后遍历
      prev = curr;
      curr = getNext(*curr);
      i++;
      continue;     
    }

    bool updateHead = activeHead_.compare_exchange_weak(prevHead, getNext(*curr));

    if(updateHead){
      if(prev){    // prev不为空，说明curr之前至少找到一个"热节点"
        unmarkAccessedBatch(prevHead, prev); // Set the state of the nodes between prevHead and curr to unAccessed.
        linkAtTail(dormantHead_, dormantTail_, *prevHead, *prev);
      }
      size_.fetch_sub(1+i);
      return curr;
    }
    curr = nullptr;    // CAS操作失败
  }
}
```
该方法用于检测和处理**连续的**已访问节点批次。
1. 它从活跃队列的头部开始，连续检查一批已访问的节点。
2. 如果它遇到第一个**未访问**的节点，它会尝试使用一次`compare_exchange_weak`原子操作，将队列头部一次性移动到这个未访问节点的下一个位置。
3. 如果操作成功，它会将之前那批连续的已访问节点一起移动到休眠队列的尾部（使用`unmarkAccessedBatch`和`linkAtTail`），然后返回当前未访问的节点作为淘汰候选项。

## 并行安全性实现
Mobius算法主要通过以下机制实现线程安全：
1. **原子变量**: `head_`, `tail_`, `size_`, `whichQ_`, `countRequest_`等关键状态都使用`std::atomic`来声明。这确保了对这些变量的读取和写入操作是原子的，不会被其他线程中断。
2. **无锁编程**: 大部分操作（如`linkAtTail`和`getEvictionCandidate`）都使用**比较并交换（CAS）操作**来实现。

## 创新特点
Mobius在传统缓存替换算法，尤其是**时钟算法和双队列(2Q)算法**的基础上进行了创新：
- **双队列结构**: 借鉴了2Q算法的思想，Mobius使用两个队列（活跃和休眠）来分离“新近访问”和“不常访问”的数据。这种结构允许算法更好地处理访问模式的变化，避免了“热”数据被过早淘汰。
- **懒惰移除**: `remove`函数只标记节点为无效，而**不立即**将其从链表中移除。实际的移除操作是在`getEvictionCandidate`遍历时才进行，这种“懒惰”清理策略简化了并发逻辑，并避免了对链表中间节点进行复杂和昂贵的原子删除操作。

