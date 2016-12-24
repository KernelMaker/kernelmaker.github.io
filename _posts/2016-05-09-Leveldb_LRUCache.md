---
layout: post
title: Leveldb之LRUCache实现
---

leveldb代码读完也有好一阵子了，其实早想好好的写博客从头到尾系统总结一遍，不过它真的是写的太好了，任何一个地方单拿出来都可以写好多，想向TCMalloc那样自下而上写一个系列的话我估计写30篇都写不完...所以就挑着一些主题写写吧，有些是和leveldb强相关的比如迭代器，compact的实现，有些是相对独立、可以用来学习编程的比如LRUCache、SkipList的实现，今天就先说说LRUCache的实现吧

## 介绍
leveldb在Table和VersionSet的实现中用到了LRUCache，分别给Block和Table加了缓存，加快访问速度，并且当缓存满了淘汰最近最少使用的那个。这篇先不管什么是Block什么是Table，就单纯的来看看levedb的LRUCache是怎么实现的

## 原理

LRUCache想必都不陌生，就是一个缓存，不过既然是缓存肯定容量有限，当缓存满了后，需要淘汰一个最近最少使用的item。怎么实现呢？也不难，无非是一个HashTable配合一个List就可以了，HashTable用来存item，List按每个item最近使用的先后顺序将所有item排序，tail的item是最近使用的，head的item是最久没使用的，要淘汰就从head开始，当用户访问完某个item，就将该item从List中拿出来再插到最尾，这样就是一个LRUCache的实现啦

## 实现

那么leveldb是怎么实现的呢？其实和上面说的差不多，不过具体实现中还是可以看出大牛的功底，一点一点来看，先说说item的实现，在leveldb中，HashTable中保存的item叫做LRUHandle，定义如下：

```cpp
struct LRUHandle {
  void* value;
  void (*deleter)(const Slice&, void* value); // 清理回调函数，用来在释放时做资源回收
  LRUHandle* next_hash; // hash冲突时线性探测
  LRUHandle* next; // List的next
  LRUHandle* prev; // List的prev
  size_t charge;      // TODO(opt): Only allow uint32_t?
  size_t key_length; //key的长度
  uint32_t refs; // 引用计数
  uint32_t hash;      // Hash of key(); used for fast sharding and comparisons
  char key_data[1];   // Beginning of key

  Slice key() const {
    // For cheaper lookups, we allow a temporary Handle object
    // to store a pointer to a key in "value".
    if (next == this) {
      return *(reinterpret_cast<Slice*>(value));
    } else {
      return Slice(key_data, key_length);
    }
  }
};
```

next_hash是在hash冲突时线性探测后续LRUHandle用到的，next和prev是记录List的节点信息，我们发现leveldb在具体实现中，将上一节说到的HashTable中的item和List中的节点揉在了一起，这样HashTable中的每个元素不仅记录着冲突hash值的下一个元素，还记录着LRU顺序

LRUHandle定义完了，接下来就是HashTable的实现了，在levledb叫HandleTable，代码如下：

```cpp
class HandleTable {
 public:
  HandleTable() : length_(0), elems_(0), list_(NULL) { Resize(); }
  ~HandleTable() { delete[] list_; }

  LRUHandle* Lookup(const Slice& key, uint32_t hash) {
    return *FindPointer(key, hash);
  }

  LRUHandle* Insert(LRUHandle* h) {
    LRUHandle** ptr = FindPointer(h->key(), h->hash); // 先在当前HandleTable中
    //查找有没有包含h->key的LRUHandle，如果有的话则返回保存该LRUHandle地址的内
    //存地址，也就是该LRUHandle的前一个LRUHandle的next字段地址
    LRUHandle* old = *ptr; //对上一步返回的ptr解引用得到对应LRUHandle地址，也
    //就是之前存在的LRUHandle地址，赋值给old
    h->next_hash = (old == NULL ? NULL : old->next_hash); // 将h的next_hash赋
    //值为old的next_hash
    *ptr = h; //将ptr的值也就是对应LRUHandle前一个LRUHandle的next字段赋值为h，
    //这样就将h插入到了桶中，old移出桶
    if (old == NULL) {
      ++elems_;
      if (elems_ > length_) {
        //当前存的元素个数大于桶的个数，扩容
        // Since each cache entry is fairly large, we aim for a small
        // average linked list length (<= 1).
        Resize();
      }
    }
    return old;
  }

  LRUHandle* Remove(const Slice& key, uint32_t hash) {
    LRUHandle** ptr = FindPointer(key, hash);
    LRUHandle* result = *ptr;
    if (result != NULL) {
      *ptr = result->next_hash;
      --elems_;
    }
    return result;
  }
  
  private:
  // The table consists of an array of buckets where each bucket is
  // a linked list of cache entries that hash into the bucket.
  uint32_t length_; // hash桶数
  uint32_t elems_; // 当前有多少个元素
  LRUHandle** list_; // 桶数组

  // Return a pointer to slot that points to a cache entry that
  // matches key/hash.  If there is no such cache entry, return a
  // pointer to the trailing slot in the corresponding linked list.
  // 通过hash值在对应桶中查找key，注意：如果存在则返回保存改LRUHandle地址的
  //内存地址，不存在则返回空
  LRUHandle** FindPointer(const Slice& key, uint32_t hash) {
    LRUHandle** ptr = &list_[hash & (length_ - 1)];
    while (*ptr != NULL &&
           ((*ptr)->hash != hash || key != (*ptr)->key())) {
      ptr = &(*ptr)->next_hash;
    }
    return ptr;
  }
  // 扩容
  void Resize() {
    uint32_t new_length = 4;
    while (new_length < elems_) {
      new_length *= 2;
    }
    LRUHandle** new_list = new LRUHandle*[new_length];
    memset(new_list, 0, sizeof(new_list[0]) * new_length);
    uint32_t count = 0;
    for (uint32_t i = 0; i < length_; i++) {
      LRUHandle* h = list_[i];
      while (h != NULL) {
        LRUHandle* next = h->next_hash;
        uint32_t hash = h->hash;
        LRUHandle** ptr = &new_list[hash & (new_length - 1)];
        h->next_hash = *ptr;
        *ptr = h;
        h = next;
        count++;
      }
    }
    assert(elems_ == count);
    delete[] list_;
    list_ = new_list;
    length_ = new_length;
  }
};

```

实现不难，不过有一点值得特殊说一下，自己理解的单向链表的Insert操作一般可以分成两步，第一步是FindPoint，来找到要插入的位置cur，第二步是在FindPoint返回的位置cur之前插入一个新的元素，这里似乎有点问题，因为一般链表的插入需要拿到cur的前一个元素并修改其next指针，但FindPoint只返回的cur，并没有返回cur的前一个元素地址，难道要返回两个指针吗？其实不用，leveldb是这么做的，FindPoint返回的并不是要插入位置元素的地址，而是保存改地址内存的地址，也就是cur前一个元素next字段的地址ptr，这样就一举两得了，通过*ptr可以找到cur，通过修改ptr->next可以直接修改前一个元素的next字段，很巧妙

好了，HandleTable实现完了，现在就可以实现LRUCache了，代码如下：

```cpp
// A single shard of sharded cache.
class LRUCache {
 public:
  LRUCache();
  ~LRUCache();

  // Separate from constructor so caller can easily make an array of LRUCache
  void SetCapacity(size_t capacity) { capacity_ = capacity; }

  // Like Cache methods, but with an extra "hash" parameter.
  Cache::Handle* Insert(const Slice& key, uint32_t hash,
                        void* value, size_t charge,
                        void (*deleter)(const Slice& key, void* value));
  Cache::Handle* Lookup(const Slice& key, uint32_t hash);
  void Release(Cache::Handle* handle);
  void Erase(const Slice& key, uint32_t hash);

 private:
  void LRU_Remove(LRUHandle* e); //修改e的next和prev指针，从lru_中移除
  void LRU_Append(LRUHandle* e); //修改e的next和prev指针，插入lru_中
  void Unref(LRUHandle* e); //减少e的引用计数，如果为0就释放e

  // Initialized before use.
  size_t capacity_;

  // mutex_ protects the following state.
  port::Mutex mutex_;
  size_t usage_;

  // Dummy head of LRU list.
  // lru.prev is newest entry, lru.next is oldest entry.
  LRUHandle lru_; //lru list的head节点，今后prev指向最新节点，next指向
  //最老节点，每次淘汰next，初始都指向自己

  HandleTable table_; //包含一个HandleTable
};

void LRUCache::LRU_Remove(LRUHandle* e) {
  e->next->prev = e->prev;
  e->prev->next = e->next;
}

void LRUCache::LRU_Append(LRUHandle* e) {
  // Make "e" newest entry by inserting just before lru_
  e->next = &lru_;
  e->prev = lru_.prev;
  e->prev->next = e;
  e->next->prev = e;
}

Cache::Handle* LRUCache::Lookup(const Slice& key, uint32_t hash) {
  MutexLock l(&mutex_);
  //从HandleTable中查找是否存在key的LRUHandler
  LRUHandle* e = table_.Lookup(key, hash);
  if (e != NULL) {
    //因为要返回给用户，所以引用数+1
    e->refs++;
    //从lru_中移除
    LRU_Remove(e);
    //插入到lru_的prev位置，这样e的淘汰优先级就最低
    LRU_Append(e);
  }
  return reinterpret_cast<Cache::Handle*>(e);
}

void LRUCache::Release(Cache::Handle* handle) {
  MutexLock l(&mutex_);
  Unref(reinterpret_cast<LRUHandle*>(handle));
}

```
实现的很清晰，重点再看一下Insert：

```cpp
Cache::Handle* LRUCache::Insert(
    const Slice& key, uint32_t hash, void* value, size_t charge,
    void (*deleter)(const Slice& key, void* value)) {
  MutexLock l(&mutex_);
  
  //LRUHandle中key_data是最后一个元素，定义是char[1]，这里有一个技巧，就是
  //定义中虽然key_data长度为1个字节，但在实际生成时，malloc的空间大小是
  //sizeof(LRUHandle)-1 + key.size()，刚好key_data是最后一个元素，这样从它
  //开始以后的空间是实际存放key的位置，对于结构体中变长字符数组的定义可以
  //学习这样的方式，很不错
  LRUHandle* e = reinterpret_cast<LRUHandle*>(
      malloc(sizeof(LRUHandle)-1 + key.size()));
  e->value = value;
  e->deleter = deleter;
  e->charge = charge;
  e->key_length = key.size();
  e->hash = hash;
  e->refs = 2;  // One from LRUCache, one for the returned handle，引用计数
  //初始为2，一个是LRUCache在用，一个是返回给用户
  memcpy(e->key_data, key.data(), key.size());
  //插入到lru_的prev位置，最低淘汰优先级
  LRU_Append(e);
  usage_ += charge;

  LRUHandle* old = table_.Insert(e); //将e插入到HandleTable中，返回的是存放
  //该key老值的LRUHandle指针
  if (old != NULL) {
    //老的LRUHandle已经没用，从lru_中移除
    LRU_Remove(old);
    //减少引用，为0时释放
    Unref(old);
  }
  //如果已经存满，则淘汰最近最少使用的LRUHandle，也就是lur_的next所指
  while (usage_ > capacity_ && lru_.next != &lru_) {
    LRUHandle* old = lru_.next;
    LRU_Remove(old); //从lru_中移除
    table_.Remove(old->key(), old->hash); //从HandleTable中移除
    Unref(old); //减少引用
  }

  return reinterpret_cast<Cache::Handle*>(e);
}
```

关键位置已经注释了，不难，至此，LRUCache已经实现差不多了，这时候已经可以结束了，不过leveldb在上面又封了一层ShardedLRUCache，其实就是包了多了LRUCache，这样每个key会定位两次，第一次是用hash值的高4位做shard，算出具体该访问哪个LRUCache，第二次同样的hash值访问LRUCache的某个桶，就是一层封装，不难，直接看实现：


```cpp
static const int kNumShardBits = 4;
static const int kNumShards = 1 << kNumShardBits;

class ShardedLRUCache : public Cache {
 private:
  LRUCache shard_[kNumShards]; // 16个LRUCache
  port::Mutex id_mutex_;
  uint64_t last_id_;

  static inline uint32_t HashSlice(const Slice& s) {
    return Hash(s.data(), s.size(), 0);
  }

  static uint32_t Shard(uint32_t hash) {
    //hash值的高4位做shard
    return hash >> (32 - kNumShardBits);
  }

 public:
  explicit ShardedLRUCache(size_t capacity)
      : last_id_(0) {
    const size_t per_shard = (capacity + (kNumShards - 1)) / kNumShards;
    for (int s = 0; s < kNumShards; s++) {
      shard_[s].SetCapacity(per_shard);
    }
  }
  virtual ~ShardedLRUCache() { }
  virtual Handle* Insert(const Slice& key, void* value, size_t charge,
                         void (*deleter)(const Slice& key, void* value)) {
    const uint32_t hash = HashSlice(key); //求hash
    return shard_[Shard(hash)].Insert(key, hash, value, charge, deleter); //插入
    //到对应的LURCache中
  }
  virtual Handle* Lookup(const Slice& key) {
    const uint32_t hash = HashSlice(key); //求hash
    return shard_[Shard(hash)].Lookup(key, hash); //在对应LRUCache中查找
  }
  virtual void Release(Handle* handle) {
    LRUHandle* h = reinterpret_cast<LRUHandle*>(handle);
    shard_[Shard(h->hash)].Release(handle);
  }
  virtual void Erase(const Slice& key) {
    const uint32_t hash = HashSlice(key);
    shard_[Shard(hash)].Erase(key, hash);
  }
  virtual void* Value(Handle* handle) {
    return reinterpret_cast<LRUHandle*>(handle)->value;
  }
  virtual uint64_t NewId() { //这个是leveldb给block_cache用的，这里可以不用关心
    MutexLock l(&id_mutex_);
    return ++(last_id_);
  }

```
  
## 总结

LRUCache看似简单，实现起来的方式确很多，本篇介绍了leveldb关于LRUCache的实现，还是非常经典的，值得学习，另外levedb主题的博客我会持续写下去，不过内容真的太多了，下篇先写写迭代器相关的东西吧^^
