---
layout: post
title: 【Rocksdb实现分析及优化】Iterator
---



今天总结下rocksdb的Iterator实现，之前的博客介绍过leveldb Iterator的实现（[传送门](http://kernelmaker.github.io/Leveldb_Iterator)），rocksdb的Iterator核心结构和leveldb差不多，不过整体实现要相对复杂一些，这么多细节要用文字描述出来恐怕得累个半死，下午整理出来两幅图，一幅是Iterator创建的大致过程，另一幅是和Iterator相关的类图



## 1. rocksdb 对Iterator的优化

相比于leveldb，我觉得rocksdb主要在Iterator的设计实现上通过以下两点来进行优化：

1. 我们知道Iterator其实是一个包含若干子Iterator树结构，rocksdb使用连续内存来创建保存所有子Iterator，这样在Iterator树中，每个Iterator指针及其指向的Iterator本身距离十分近，这样更加cache友好，提高效率。在具体实现上就是从最开始创建Iterator树时，将全局的Arena不断向下传递，使用Arena来分配内存，这样便能达到使用连续内存的目的
2. leveldb的MergeIterator每次在Next或Prev时，会遍历其所有的子Iterator来选出它们中最小或最大的Iterator进行读取，这里选择采用的是遍历的方法，复杂度即O(n)。rocksdb的MergeIterator则通过Heap来维护所有的子Iterator，每次选出最小值后整理的复杂度为O(lgn)。另外Next对应的是MinHeap，Prev对应的是MaxHeap，MinHeap是在MergeIterator创建时就构造好，而MaxHeap则是在需要的时候才构造，因为一般用户使用Next的概率远高于Prev，这么做可以节约内存



## 2. Iterator关系及创建过程

主要关系及创建过程都在下图中，此处有几点说明：

1. ArenaWrappedDBIter是暴露给用户的Iterator，它包含DBIter，DBIter则包含InternalIterator，InternalIterator顾名思义，是内部定义，MergeIterator、TwoLevelIterator、BlockIter、MemTableIter、LevelFileNumIterator等都是继承自InternalIterator
2. 图中绿色实体框对应的是各个Iterator，按包含顺序颜色由深至浅
3. 图中虚线框对应的是创建各个Iterator的方法
4. 图中蓝色框对应的则是创建过程中需要的类的对象及它的方法

<img src="/public/images/2017-04-09/1.png" width="800px" />

## 3. Iterator类图

这不是一个标准的类图，不过就和我总结ColumnFamily那张一样，主要是为了理清各个类之间的关系，此处有几点说明：

1. 可以看到对内和对外的Iterator类定义是分开的（只是有一个公共父类Cleanable），对外Iterator包含对内Iterator

2. 实线空心箭头代表继承

3. 实线实心箭头代表类似包含的关系

4. 虚线实心箭头：类似间接包含关系，因为MergeIterator中并不是直接包含各个子Iterator（MemTableIterator，TwoLevelIterator），而是通过IteratorWrapper把每个子Iterator包了一层，所以这里应该还有一个虚线实心箭头从MergeIterator连到MemtableIterator，只不过画不下了，特此说明。（我的画图布局能力真是汗\|\|\|）

   ​

<img src="/public/images/2017-04-09/2.png" width="800px" />

## 4. 总结

终于整理完Rocksdb的Iterator了，这样rocksdb的几个大的块（读，写，迭代，compaction、flush）基本就搞定了，剩下的都是一些小点了，慢慢搞^^