---
layout: post
title: 【Rocksdb实现分析及优化】level_compaction_dynamic_level_bytes
---



今天总结下rocksdb的level_compaction_dynamic_level_bytes



## 1. 空间放大

空间放大是rocksdb几乎逃避不了的问题，假设某业务场景如下：

> 1. keys总数恒定，每天会全量更新所有keys
> 2. DB 大小恒定，所有key-value一共28G



以rocksdb默认配置来分析，每一层大小限制如下：

> Level 1 —> 256M
>
> Level 2 —> 2.56G
>
> Level 3 —> 25.6G
>
> Level 4 —> 256G
>
> ...

如果要求空间放大为0，即rocksdb中都是有效数据，则应该在全量更新数据后，Level 0 到 Level 3每一层都为0，28G数据全部都在Level 4，如下：

> Level 1 —> 0M
>
> Level 2 —> 0M
>
> Level 3 —> 0M
>
> Level 4 —> 28G
>
> ...

而想要达到这样的结果，则应该在某次全量更新结束后执行CompactRange来主动进行全局Compaction，此时空间放大为0，十分理想

第二天同样继续来一波28G的全量更新，如果不强制执行compactRange，那此时每一层的大小有可能是这样：

> Level 1 —> 250M
>
> Level 2 —> 2.55G
>
> Level 3 —> 25.2G
>
> Level 4 —> 28G (前一天的全量数据)

此时rocksdb刚好有两天的全量数据，Level 1 到 Level 3 总和28G是今天的，Level 4的28G是昨天的，此时写放大是(28G + 28G) / 28G = 2，整整多占了一倍的磁盘空间，并且AutoCompaction无能为力，我们明知道如果将Level 1 到 Level 3的数据推到Level 4，则昨天的数据会全部被Drop掉，只剩下今天的数据在Level 4，磁盘占用会才从56G减少到28G，但按照rocksdb默认的compaction策略，Level 1到 Level 3都没有达到触发AutoCompaction的大小。

怎么解决呢，我们知道rocksdb默认是base_level是Level 1，大小上限是256M，然后每一层大小上限基于此乘以10，依次往下，对于上述情况，能否改变思路，不要从上往下来定每一层的大小上限，而是从下往上来定，这样倒着搞更有利于整体保持正三角的形状，而正三角的写放大差不多是1.1111，还是十分理想的。



## 2. level_compaction_dynamic_level_bytes

### 1. 原理

如果打开level_compaction_dynamic_level_bytes，则base_level会从默认的Level 1 变成最高层 Level 6，即最开始Level 0会直接compact到Level 6，如果某次compact后，Level 6大小超过256M(target_file_size_base)，假设300M，则base_level向上调整，此时base_level变成Level 5，而Level 5的大小上限是300M/10 = 30M，之后Level 0会直接compact到Level 5，如果Level 5超过30M，假设50M，则需要与Level 6进行compact，compact后，Level 5恢复到30M以下，Level 6稍微变大，假设320M，则基于320M继续调整base_level，即Level 5的大小上限，调整为320M/10 = 32M，随着写入持续进行，最终Level 5会超过256M(target_file_size_base)，此时base_level需要继续上调，到Level 4，取Level 5和Level 6当前大小较大者，记为MaxSize，则Level 4的大小上限为MaxSize/100，Level 5的大小上限为Level 4大小上限乘以10，依次类推。

### 2. 实现

在rocksdb中：每次base_level及其大小上限(base_bytes)的调整发生在LogAndApply之后，根据当前每一层的现状来进行调整，实现逻辑在VersionStorageInfo::CalculateBaseBytes()中，大致过程如下：

> 1. 从first_non_empty_level到最后一层，统计每一层的大小，找出最大者，记为max_level_size，最大者不一定是最后一层，因为有可能在某次compaction后，其他层的大小会大于最后一层
> 2. 从倒数第二层往上到first_non_empty_level，假设有n层，则cur_level_size = max_level_size/(10^n)，cur_level_size是当前first_non_empty_level的新的大小上限
> 3. 如果cur_level_size > max_bytes_for_level_base(256M)，则对cur_level_size除以10继续向上调整first_non_empty_level，直到调整到某一层，cur_level_size <= max_bytes_for_level_base(256M)，此时该层为新的base_level，即新的first_non_empty_level，base_size为cur_level_size
> 4. 然后从base_level开始，往下，每一层对base_size乘以10，当做该层新的大小限制



## 3. 总结

很好的减少空间放大的办法，值得学习