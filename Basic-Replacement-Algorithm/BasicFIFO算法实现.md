## 算法级实现
对于高并发场景下的缓存替换算法，性能瓶颈之一是互斥锁的资源消耗。
这里首先实现一个最基础的线程安全的FIFO替换算法：使用全局互斥锁，unordered_set提供常数级查找缓存
```C++
#include <iostream>
#include <queue>
#include <unordered_set>
#include <mutex>
#include <thread>
#include <vector>

class ThreadSafeFIFOCache
{
public:
    ThreadSafeFIFOCache(size_t capacity) : capacity_(capacity) {}

    // 访问缓存数据
    bool access(int key)
    {
        std::lock_guard<std::mutex> lock(mutex_);

        // 缓存命中
        if (cacheSet_.count(key))
        {
            std::cout << "Cache hit: " << key << std::endl;
            return true;
        }

        // 缓存未命中，替换
        if (cacheQueue_.size() == capacity_)
        {
            int oldest = cacheQueue_.front();
            cacheQueue_.pop();
            cacheSet_.erase(oldest);
            std::cout << "Evict: " << oldest << std::endl;
        }

        cacheQueue_.push(key);
        cacheSet_.insert(key);
        std::cout << "Cache miss, insert: " << key << std::endl;
        return false;
    }

private:
    size_t capacity_;                  // 缓存容量大小
    std::queue<int> cacheQueue_;       // 维护FIFO顺序
    std::unordered_set<int> cacheSet_; // 快速判断是否存在缓存
    std::mutex mutex_;                 // 保护共享数据
};

// 模拟并发访问
void worker(ThreadSafeFIFOCache &cache, const std::vector<int> &requests)
{
    for (int key : requests)
    {
        cache.access(key);
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
    }
}

int main()
{
    ThreadSafeFIFOCache cache(3);

    // 两个线程模拟不同访问请求
    std::thread t1(worker, std::ref(cache), std::vector<int>{1, 2, 3, 4, 1});
    std::thread t2(worker, std::ref(cache), std::vector<int>{2, 5, 1, 6});

    t1.join();
    t2.join();

    return 0;
}

```
在main函数中定义了容量大小为3的缓存池，两个线程模拟并行访问。