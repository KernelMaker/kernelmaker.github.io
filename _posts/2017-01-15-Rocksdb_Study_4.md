---
layout: post
title: 【Rocksdb实现分析及优化】 Write及Flush实现
---

最近缕了一下Rocksdb的Write和Flush逻辑，实现的非常细致，外加ColumnFamily，导致这块代码逻辑巨多，还好，这两天把这块的东西缕出来画了两幅图，方便以后可以快速回忆

## 图注释

### 1. 类图

这个不是标准的UML图，简单给自己看的

* 细实心箭头实线代表类似包含的关系
* 粗实心箭头实线代表包含多个
* 细空心箭头实线代表继承
* 细实心箭头虚线代表反向依赖

### 2. 流程图

* 子节点是父节点函数的一部分
* 菱形switch代表这里是多选一的情况
* 平行四边形代表是switch的一种情况表述，后跟本情况下的操作
* 红色实心箭头虚线代表指向具体实现
* 橙色字体主要是标注mutex_的Lock和Unlock，方便辨识
* 绿色字体主要是标注MaybeScheduleFlushOrCompaction，方便辨识



## 图

### 1. 类图

<img src="/public/images/2017-01-15/1.png" width="1000px" />



### 2. 流程图

<img src="/public/images/2017-01-15/2.png" width="1000px" />



## 总结

Write和Flush实现的非常细致，后续再缕一下Compact和Iterator