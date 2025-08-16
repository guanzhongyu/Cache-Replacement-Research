## 模板类的定义
### ClockListHook 类 
这个结构体包含三个主要成员：
1. **`CompressedPtr next_`**: 指向链表中**下一个**节点的压缩指针。
2. **`CompressedPtr prev_`**: 指向链表中**上一个**节点的压缩指针。
3. **`uint32_t updateTime_`**: 记录节点最后一次被更新到链表头部的时间戳。

`ClockListHook` 的设计遵循了**侵入式（intrusive）链表**的原则。传统的非侵入式链表（如 `std::list`）通常会为每个节点动态分配额外的内存，并存储指向用户数据的指针。相比之下，侵入式链表将**链表节点本身的数据**直接嵌入到用户对象中。
这种设计有几种关键优势：
- **内存效率**: 无需额外的内存分配。每个缓存对象都自带了链表所需的 `next` 和 `prev` 指针，从而节省了大量内存开销，这对于存储数十亿个小型对象的缓存系统至关重要。
- **缓存友好**: 由于链表节点和用户数据在内存中是连续的，访问链表时可以更好地利用 CPU 缓存，提高性能。
- **通用性**: 通过模板化设计，`ClockListHook` 可以嵌入到任何想要加入 `ClockList` 的对象中，实现了算法与数据存储的分离。

### ClockList 类 
它通过一个**循环双向链表**和每个对象的“访问位”，实现了以下功能：
- **追踪访问**: 当一个对象被访问时，只通过一个简单的位操作（设置访问位），避免了昂贵的链表移动操作（相较于LRU算法）。
- **高效淘汰**: 当需要释放内存时，它从链表尾部开始遍历，通过检查访问位来快速找到一个最不常使用的对象进行淘汰，而不是盲目地移除链表末尾的元素。

ClockList 类 内部主要包含以下内容：
1. 成员变量：
	1. `head_`, `tail_`: 指向链表头尾的原子指针。
	2. `size_`: 链表中元素的原子计数。
	3. `compressor_`: 用于压缩和解压缩指针，以节省内存。
	4. `head_mutex`, `mtx_`: 用于保护并发访问的互斥锁。
2. 成员方法：
	1. **`linkAtHead(T& node)`**: 将一个节点添加到链表头部。
	2. **`removeTail()`**: 移除链表尾部的一个节点，并返回它。
	3. **`getEvictionCandidate()`**: **时钟算法的核心实现**，从链表尾部开始寻找一个可以淘汰的候选对象。
	4. **`markAccessed(T& node)`**: 设置节点的访问位。
	5. **`unmarkAccessed(T& node)`**: 清空节点的访问位。

## 设计思路
```C++
template <typename T, ClockListHook<T> T::*HookPtr>
T* ClockList<T, HookPtr>::getEvictionCandidate() noexcept {
  T* curr = nullptr;

  while (true) {
    curr = removeTail();
    if (curr == nullptr) {
      return nullptr;
    }
    if (isAccessed(*curr)) {
      unmarkAccessed(*curr);
      linkAtHead(*curr);
    } 
    else {
      return curr;
    }
  }
}
```


假设我们有一个由 `A`、`B`、`C`、`D` 四个缓存项组成的循环双向链表。
`D` (尾部) <-> `C` <-> `B` <-> `A` (头部)
`getEvictionCandidate()` 方法会从链表的尾部（`D`）开始，执行以下步骤：
1. **移除尾部**:首先调用 `removeTail()`，将 `D` 从链表尾部移除。此时链表变为：`C` <-> `B` <-> `A`，新的尾部是 `C`。
2. **检查访问位**: 检查 `D` 的访问位（通过 `isAccessed(*D)`）。
	1. 场景 1：`D` 的访问位为 `true`
		1. **给予二次机会**: 这意味着 `D` 最近被访问过。
		2. **清空标记**: 调用 `unmarkAccessed(D)` 清除其访问位。
		3. **重新入队**: 调用 `linkAtHead(D)` 将 `D` 重新添加到链表的**头部**。
		4. **继续循环**: 时钟指针继续寻找下一个淘汰对象，即新的尾部 `C`。
	2. 场景 2：`D` 的访问位为 `false`
		1. **直接淘汰**: 这意味着 `D` 在上次时钟指针经过后没有被访问过。
		2. **结果**: 此时 `D` 已被移除，链表只剩下 `C` <-> `B` <-> `A`。

## 安全一致性保障
并发的原子性与操作的安全一致性主要通过以下三个方面来保证：
1. 原子变量：
	1. 使用了 `std::atomic` 模板类来声明关键的链表指针和大小计数器。原子变量保证了对这些变量的读写操作是**不可分割**的，不会被其他线程中断。
		1. **`std::atomic<T*> head_`**: 链表的头指针。
		2. **`std::atomic<T*> tail_`**: 链表的尾指针。
		3. **`std::atomic<size_t> size_`**: 链表的元素数量。

> 以`linkAtHead` 方法为例，它通过 `head_.compare_exchange_weak(oldHead, &node)` 循环来更新头指针：
> 1. 它首先读取当前的 `head_` 到 `oldHead`。
> 2. 然后，它尝试将 `head_` 的值从 `oldHead` 更新为新节点 `&node`。
> 3. 如果在此期间没有其他线程修改 `head_`，操作成功。
> 4. 如果其他线程先一步修改了 `head_`，`compare_exchange_weak` 会失败，并将新的 `head_` 值写回 `oldHead`。循环会再次执行，尝试用最新的 `head_` 值进行更新。
> 这种“比较并交换”（Compare-and-Swap, **CAS**）的自旋操作确保了在不加全局锁的情况下，多个线程可以安全地竞争更新链表头，保证了最终结果的正确性。

2. 互斥锁 (Mutex)：对于一些需要进行多步操作、不能用单个原子指令完成的复杂逻辑，代码使用了互斥锁 (`Mutex`) 来保证临界区的排他性访问。
3. 自旋锁机制：`getEvictionCandidate()` 方法内部的 `while(true)` 循环，结合 `removeTail()` 方法中的 `tail_.compare_exchange_weak`，形成了一种**自旋锁**的机制。
	1. **`removeTail()`** 方法通过 CAS 操作来原子性地更新 `tail_`。如果更新失败，它会重新读取 `tail_` 的最新值，并再次尝试。这避免了在获取尾部元素时需要加锁，提升了性能。
	2. `getEvictionCandidate()` 本身是一个 `while(true)` 循环，它不断地尝试从尾部移除元素。这个循环会一直“自旋”，直到成功找到一个未被访问过的元素。它利用了乐观锁的思想，假设冲突很少发生，只在冲突发生时才重试，而不是一开始就加锁。