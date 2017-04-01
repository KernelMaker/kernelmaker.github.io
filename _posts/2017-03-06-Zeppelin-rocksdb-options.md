---
layout: post
title: zeppelin使用场景下的rocksdb配置
---



不管是pika还是bada，在之前的开发中，几乎对rocksdb的默认options没做什么改动，基本都是在无脑用的状态，zeppelin也快要上线，由于zeppelin的使用场景比较特殊，这两天看了下rocksdb大部分options的实现，对这块有了较深入的了解，所以结合zeppelin，尝试给出一套推荐的配置

## 1. Zeppelin使用环境

zeppelin较之pika的使用环境，主要有如下不同：

1. 运行在容量较大的机械盘
2. 一台机器上会开几十个rocksdb实例
3. 单条value较大，约512K ~ 1M



## 2. 机械盘下有何不同

rocksdb在机械盘环境下，较SSD有何不同：

1. 机械盘容量更大，这就意味着单盘可存储的数据更多，而此时内存却没有随之变大，所以有内存不够用的风险，要通过修改options合理控制内存使用
2. 机械盘随机读性能变差，并且随机读与顺序读差异更明显，所以要尽可能的减少文件读的次数，尽可能使用顺序读来替代随机读



## 3. 推荐配置

假设机器内存64G，磁盘容量24T（24块1T盘），单台机器开48个rocksdb实例（一块盘开两个rocksdb实例，即两个zeppelin的partition），单条value 512K，则有如下配置：

```c++
Options base_options;

// memtable大小256M，48个实例最多占用256M*2*48 = 24.6G
base_options.write_buffer_size = 256 * 1024 * 1024;
// sst大小256M，这样可以减少sst文件数，从而减少fd个数
base_options.target_file_size_base = 256 * 1024 * 1024;
/*
 * level 1触发compaction的大小为128M*4 = 512M
 * （这里假设从memtable到level 0会有50%的压缩）
 */
base_options.max_bytes_for_level_base = 512 * 1024 * 1024;
// 加快db打开的速度
base_options.skip_stats_update_on_db_open = true
/*
 * compaction对需要的sst单独打开新的句柄，与Get()互不干扰
 * 并且使用2M的readhead来加速read，这里分配2M会带来额外的内存
 * 开销，默认单次compaction涉及的最大bytes为
 * target_file_size_base * 25，即25个sst文件，则每个rocksdb
 * 实例会额外消耗25*2M = 50M，48个实例一共消耗50M*48 = 2.4G
 */
base_options.compaction_readahead_size = 2 * 1024 * 1024;
/*
 * 24T数据大约有198304个sst(256M)文件，则48个rocksdb实例
 * 每一个实例差不多有2048个，所以配置table_cache的capacity
 * 为2048
 */
base_options.max_open_files = 2048;

BlockBasedTableOptions block_based_table_options;
/*
 * 使用512K的block size，修改block_size主要是为了减少index block的大小
 * 但鉴于本例中单条value很大，其实效果不明显，所以这个可改可不改
 */
block_based_table_options.block_size = 512 * 1024;

base_options.table_factory.reset(
  NewBlockBasedTableFactory(block_based_table_options));
```

以上差不多就是最需要改动的options了，前期使用这样的配置在线上跑，可以看到在不计table_cache内存消耗的情况下，以上配置会占用28G左右内存，不过table_cache向来都是内存占用大户，所以下面的配置项则可以根据线上实际效果酌情修改：

#### 1. 发现compaction造成磁盘io较高

```c++
//可以打开它来减少写放大
base_options.level_compaction_dynamic_level_bytes = true;
```

level_compaction_dynamic_level_bytes我会在后续的博客中详细介绍

#### 2. 内存不够用

内存不够用，假设24T盘全部存上数据，则区区64G内存显然太小，这时候只能靠牺牲部分性能来降内存，rocksdb的内存占用大户有一个table_cache、里面缓存这index和filter block，通过下面的配置来减少他们

```c++
//最后一层不要filter
base_options.optimize_filters_for_hits = true;
//除level 0之外其他文件的index和filter都缓存在block_cache
block_based_table_options.cache_index_and_filter_blocks = true;
```

block_cache默认大小8M，48个实例占用384M，假设打开上面的配置来大幅度减少table_cache的内存消耗后，384M的block_cache肯定存不了多少index及filter_blocks，这时候可以适当增加block_cache的大小，单个block_cache设置成32M，则一共可以使用1.5G来进行block的缓存

#### 3. 线程数过多

zeppelin本身就开了很多线程，48个rocksdb实例，每个实例2个线程（1个flush，1个compaction），那么这就是96个线程，可以通过全局使用同一个options来共享线程，可以配置成这样

```cpp
base_options.max_background_flushes = 24;
base_options.max_background_compactions = 24;
```

这样，通过将同一个options传给48个rocksdb的open()，来共享这48个线程。另外由于block_cache是在options中，如果要共享options的话，block_cache也是共享的，所以block_cache应该从32M改成1.5G



## 总结

rocksdb在细节处做了非常多的优化和配置，值得学习。