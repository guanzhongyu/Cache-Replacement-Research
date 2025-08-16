## 实现原理：
1. **`SieveListHook`**:
	1. 这是一个**侵入式**的钩子结构，包含 `next_` 和 `prev_` 指针，用于构建双向链表。
	2. 它不包含访问位，访问位是通过 `T::Flags::kMMFlag1` 来实现的。这意味着访问位是缓存项 `T` 本身的一个标志，而不是 `SieveListHook` 的成员。
2. `SieveList`:
	1. `head_`, `tail_`: 链表的头尾指针。新项通过 `linkAtHead()` 加入链表。
	2. `curr_`: **筛子指针**。这是该算法的核心状态。它不是每次都从头开始，而是从上一次停止的地方继续。
	3. `getEvictionCandidate()`: 算法的核心实现。它从 `curr_` 指针开始，通过一个循环遍历链表。
		1. 在循环中，它检查当前节点（`*curr_`）的 `isAccessed` 标志。
		2. 如果标志为 `true`，它会调用 `unmarkAccessed` 清除该标志，然后将 `curr_` 指针移动到下一个节点 (`getNext(*curr_)`)。
		3. 如果标志为 `false`，则说明找到了一个可淘汰的项。它会**将该项从链表中移除**（`remove`），然后返回。
		4. 当 `curr_` 到达 `nullptr`（即链表末尾）时，它会重置为 `head_`，继续循环。

## 并发控制：
1. **互斥锁**: `SieveList` 依赖一个互斥锁 `mtx_` 来保证并发操作的原子性和一致性。
	- `linkAtHead`, `remove`, `replace`, 和 `getEvictionCandidate` 这些方法都使用了 `LockHolder l(*mtx_)` 来获取锁，确保同一时间只有一个线程能执行这些修改链表结构的操作。
2. **原子变量**: `head_`, `tail_`, `curr_`, 和 `size_` 都是原子变量，用于保证简单的读写操作是线程安全的。

## 淘汰算法：getEvictionCandidate()

```C++
template <typename T, SieveListHook<T> T::*HookPtr>
T* SieveList<T, HookPtr>::getEvictionCandidate() noexcept {
  if (size_.load() == 0)
    return nullptr;

  LockHolder l(*mtx_);

  int n_iters = 0;
  T* ret = nullptr;

  T* curr = curr_.load();
  T* next;

  while (ret == nullptr) {
    if (curr == head_.load()) {
      curr = tail_.load();
      if (n_iters++ > 2) {
        printf("n_iters = %d\n", n_iters);
        abort();
      }
    }
    if (isAccessed(*curr)) {
      unmarkAccessed(*curr);
      curr = getPrev(*curr);

      /******* clock ********/
      // next = getPrev(*curr);
      // // move the node to the head of the list
      // moveToHead(*curr);
      // curr = next;
      /******* clock ********/
    } else {
      next = getPrev(*curr);
      ret = curr;
      unlink(*curr);
      setNext(*curr, nullptr);
      setPrev(*curr, nullptr);
      curr = next;
    }
  }
  curr_.store(curr);

  return ret;
}
```

1. **获取锁**: 首先通过 `LockHolder l(*mtx_)` 获取锁，确保在整个淘汰过程中，链表结构不会被其他线程修改。
2. **初始化**:
	1. 如果列表为空，直接返回 `nullptr`。
	2. 将 `curr_` 指针加载到局部变量 `curr` 中，作为筛子开始的位置。
3. **循环筛选**: 进入一个 `while (ret == nullptr)` 循环，持续寻找可淘汰的项。
	1. **边界处理**: `if (curr == head_.load()) { curr = tail_.load(); }` 这段逻辑是 SIEVE 算法与时钟算法的关键区别。筛子指针 `curr` 从链表尾部向头部移动，当它到达头部时，会**跳回到尾部**，然后继续向头部移动，形成一个**折返式**的遍历路径。
4. **淘汰与返回**:
	1. 将找到的项赋值给 `ret`。
	2. 从链表中**移除**该项 (`unlink(*curr)`)，并清除其指针。
	3. 更新 `curr` 指针到前一个项，以便下次淘汰时从这里开始。
	4. 退出循环，返回淘汰的项 `ret`。

## 总结
在 `SieveList::linkAtHead()` 方法中：
```C++
void SieveList<T, HookPtr>::linkAtHead(T& node) noexcept {
  LockHolder l(*mtx_);

  setPrev(node, nullptr);

  T* oldHead = head_.load();
  setNext(node, oldHead);

  while (!head_.compare_exchange_weak(oldHead, &node)) {
    setNext(node, oldHead);
  }

  ...

  size_++;
}
```
这段代码的核心逻辑是：
1. 它使用 `setNext(node, oldHead)` 将新节点 `node` 的下一个指针指向当前的链表头部 `oldHead`。
2. 然后，它通过 `head_.compare_exchange_weak(oldHead, &node)` 尝试原子地将链表头部指针 `head_` 更新为新节点 `node`。
这个过程确保了新的缓存项总是被放在链表的头部。

在getEvictionCandidate方法中：
1. 每次遍历都是从链表的尾部向头部遍历，尾部存储的缓存项是不常使用的。