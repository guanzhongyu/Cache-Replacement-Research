```C++
#include <iostream>
#include <unordered_map>
#include <list>

template<typename Key, typename Value>
class BasicLRUCache {
public:
    BasicLRUCache(size_t capacity) : capacity_(capacity) {}

    // 获取值，如果 key 不存在返回 false
    bool get(const Key& key, Value& value) {
        auto it = cacheMap_.find(key);
        if (it == cacheMap_.end()) return false;

        // 最近使用的条目移动到链表头
        cacheList_.splice(cacheList_.begin(), cacheList_, it->second);
        value = it->second->second;
        return true;
    }

    // 插入或更新值
    void put(const Key& key, const Value& value) {
        auto it = cacheMap_.find(key);
        if (it != cacheMap_.end()) {
            // 更新值并移动到链表头
            it->second->second = value;
            cacheList_.splice(cacheList_.begin(), cacheList_, it->second);
        } else {
            // 插入新节点
            if (cacheList_.size() >= capacity_) {
                // 删除最久未使用的
                auto last = cacheList_.back();
                cacheMap_.erase(last.first);
                cacheList_.pop_back();
            }
            cacheList_.emplace_front(key, value);
            cacheMap_[key] = cacheList_.begin();
        }
    }

private:
    size_t capacity_;
    std::list<std::pair<Key, Value>> cacheList_;  // 双向链表，按访问顺序排列
    std::unordered_map<Key, typename std::list<std::pair<Key, Value>>::iterator> cacheMap_;
};

```