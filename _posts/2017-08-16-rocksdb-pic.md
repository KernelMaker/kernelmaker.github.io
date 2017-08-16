---
layout: post
title: 【Rocksdb实现分析及优化】画了几张rocksdb的图
---

这两天借着部门分享，做了几张rocksdb的图，放博客里吧。



### 1. 整体图

<img src="/public/images/2017-08-16/1.png" width="500px" />



## 2. 层级图

<img src="/public/images/2017-08-16/2.png" width="500px" />



## 3. SST结构

<img src="/public/images/2017-08-16/3.png" width="500px" />

每个sst文件打开是一个TableReader对象，它会缓存在table_cache中，这个对象里包含了meta block和index block，所以table_cache基本上也是这两个东西占用的内存。

默认index block是一整块，如果使用了partitioned index，那么index block会切分成很多小块，并且建立一个二级索引，索引块在index block的最后一块，如上图。
