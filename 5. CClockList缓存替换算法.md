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

### 迭代器类Iterator
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

## 双向链表的基础操作具体实现
