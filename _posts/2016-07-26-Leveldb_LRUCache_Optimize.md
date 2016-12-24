---
layout: post
title: Leveldb之对LRUCache优化
---

昨天看到leveldb近期内有一条commit：`fix problems in LevelDB's caching code`，话说之前自己有一篇博客总结了leveldb cache的实现，既然这次它有了这方面的更新，那么就接着写一篇，总结一下cache方面的更新吧

## 回顾
这个是自己之前总结的leveldb cache的实现：[传送门](http://kernelmaker.github.io/2016/05/09/Leveldb_LRUCache.html)

## 原有cache的问题：

首先看一下官方如何描原有cache存在的问题：

### The observed behaviour

If LevelDB at any time used more file handles concurrently than
the cache size set via `max_open_files`, it attempted to reduce the
number by evicting entries from the table cache.  This could
happen most easily during compaction, and if `max_open_files` was
low.  Because the handles were in use, their reference count did
not drop to zero, and so the usage sum in the cache was not
modified by the evictions.  Subsequent Insert() calls returned
valid handles, but their entries were immediately evicted from
the cache, which though empty still acted as though full.  As a
result, there was effectively no caching, and the number of open
file handles rose []ly until it hit system-imposed limits and
the process died.

If one set `max_open_files` lower, the cache was more likely to
exhibit this beahviour, and cause the process to run out of file
descriptors.  That is, `max_open_files` acted in almost exactly the
opposite manner from what was intended.


### The problems


#### 1
The cache kept all elements on its LRU list eligible for capacity eviction---even those with outstanding references from clients.  This was ineffective in reducing resource consumption because there was an outstanding reference, guaranteeing that the items remained.  A secondary issue was that there is no guarantee that these in-use items will be the last things reached in the LRU chain, which actually recorded "least-recently requested" rather than "least-recently used".

#### 2
The sum of usages was decremented not when a (key,value) was evicted from the cache, but when its reference count went to zero.  Thus, when things were removed from the cache, either by garbage collection or via Erase(), the usage sum was not necessarily decreased.  This allowed the cache to act as though full when it was in fact not, reducing caching effectiveness, and leading to more resources being consumed---the opposite of what the evictions were intended to achieve.

#### 3
(minor) The cache's clients insert items into it by first looking up the key, and inserting only if no value is found.  Although the cache has an internal lock, the clients use no locking to ensure atomicity of the Lookup/Insert pair.  (see `table/table.cc`:  `block_cache->Insert()` and `db/table_cache.cc`:  `cache_->Insert()`).  Thus, if two threads Insert() at about the same time, they can both Lookup(), find nothing, and both Insert().  The second Insert() would evict the first value, leaving each thread with a handle on its own version of the data, and with the second version in the cache.  It would be better if both threads ended up with a handle on the same (key,value) pair, which implies it must be the first item inserted.  This suggests that Insert() should not replace an existing value.

### 小结
官方描述说着这么多，其实我觉得核心就一个问题，就是关于cache的usage回收时机不当的问题：capacity是cache的设定大小，usage是当前已使用的大小，当usage大于capacity时则触发淘汰，淘汰lru链中最老一些的元素直到usage降下来，不过如果淘汰的元素在被淘汰时还在被外部引用（用户使用中），则虽然该元素会从cache中被踢出，但其占用的usage并没有释放，直到该元素真正被释放时（元素引用数变为0时），其占用的usage才被回收，这样当capacity不大并且缓存的大量元素迟迟不被外部释放时，会导致cache的usage一直大于capacity而持续进行淘汰

这个问题的解决也十分简单，就是调整一下usage的回收时机，当发生淘汰并且被淘汰元素仍在被外部引用时，那么将其从cache中踢出的同时标注其已不再cache中并释放其占用的usage，这样usage的的回收会更为及时

## 改进实现

之前的实现就是cache的标准实现，一个HandleTable(`table`)加一个List(`lru_`)，此次更新之后的cache实现，在此基础之上多了一个List(`in_use_`)，实际上它是分担了之前List(`lru_`)的部分功能，用来区分哪些当前哪些item是用户正在使用(`in_use_`)，哪些item用户已经不用并存在于缓存之中（`lru_`)，一个item如果还在cache中，那么它要么在`in_use_`中要么在`lru_`中，不可能同时存在于二者，下面直接来看下实现

### 主要结构
Item:

	1. refs：引用计数，首次insert后为2（包含cache自身引用和外部引用）
	2. in_cache：bool，标记此Item是否在cache中

Cache:

	1. HandleTable
	2. lru：此链表中存着仅被cache自身引用的Item
	3. in_use：此链表中存着正在被外部引用的Item

item是存在cache中的元素，它自身属性主要包括refs（引用计数）和`in_cache`（当前是否在cache中）

### 内部接口

#### Ref
* 如果之前item在`lru_`中(refs==1 && `in_cache`==true)，将其移动到`in_use_`中
* refs++

#### Unref

* refs--
* IF refs==0:
	* 彻底释放item
* ELSE IF item不再被用户引用（refs==1 && `in_cache` == true)：
	* 将其从`in_use_`中移动到`lru_`中

Ref和Unref都是内部接口，给insert及release来调用的

### 外部接口

#### Insert

* 如果当前cache还有空闲容量（capacity > 0)：
	* 创建新的item，设置其refs = 2，将它插入到`in_use_`中，增大相应usage（代表cache目前已使用多少容量）
	* 将item插入到`table`中，此时分两种情况：
		* 1、之前`table`中存在相同key的item，将其从`table`中用当前item置换出来，并且将老的item从cache中释放（从其List，`lru_`或者`in_use_`中删除，减少相应usage，Unref）
		* 2、之前`table`中不存在相同key的item，直接插入即可
* 如果当前cache已满（usage > capacity)并且`lru_`不为空（有目前没有正在被使用的缓存item）：
	* 从`lru_`中取出最老的缓存item，将其释放（从`lru_`中删除，减少相应usage，Unref）

#### Lookup

* 从cache中查找相应的item
* 如果找到，Ref：
	
有一点需要特殊说明一下，关于在lookup中item此时存在与哪里（`lru_`或`in_use_`)是如何判断的，如下：
	
* item 在cache中：
	 * item在`lru_`中：refs==1(cache自身引用) && `in_cache` == true
	 * item在`in_use_`中：refs>1(cache自身引用+用户引用） && `in_cache` == true
* item 不在cache中：
 	 * refs>=1(用户引用） && `in_cache` == false

#### Release
* Unref

Release接口是给用户调用的，用来显式解除用户对item的引用

#### Erase

* 从cache中将对应item释放（从其List，`lru_`或者`in_use_`中删除，减少相应的usage，Unref）

Erase接口与Release接口互补，用来从cache中移除item并解除cache对item的引用

### 总结

这个cache中的问题自己之前看代码并没有察觉到，可能还是在看的时候思考不够深入吧，其实再小的问题都值得琢磨，还是要好好学习呀^^
