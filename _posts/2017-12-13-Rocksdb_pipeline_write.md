---
layout: post
title: 【Rocksdb实现分析及优化】enable_pipelined_write
---

rocksdb v5.5.1的配置项中多了一个enable_pipelined_write，RELEASE中描述如下：

> New option enable_pipelined_write which may improve write throughput in case writing from multiple threads and WAL enabled.

就是靠pipeline来进一步提高写性能。



## 1. 哪里可以pipeline

rocksdb的多线程写入流程之前的博客详细说过，大致如下：

1. 多线程的Write操作会首先入队列WriteThread::newest_writer_，入队之后如果发现自己是leader（队头），则继续如下操作，如果不是leader是follower，则阻在WriteThread::AwaitState上
2. leader将当前队列中所有的writer打入一个batch，然后写WAL和Memtable
3. leader更新batch中的所有writer的state，然后选择当前队列中除batch外第一个writer当新的leader，唤醒新leader和batch中的followers
4. 阻塞在WriteThread::AwaitState上的followers都被唤醒，发现自己的state已经被更新，直接返回



上面的步骤中，哪里可以优化呢？

如果将写WAL和Memetable分开，leader只负责写WAL，写完之后就唤醒新的leader，然后把当前batch插入到一个新的队列中，这个队列专门负责写Memtable，写完以后唤醒batch中的follower和该队列中的下一个leader。

这样就形成了一个pipeline，进一步提高性能。



## 2. 实现

实现也不难，除之前WriteThread::newest\_writer\_队列外，新增了一个WriteThread::newest\_memtable\_writer\_队列，负责上面说的排队写Memtable。

下面简单记录下代码，方便回顾：

```cpp
Status DBImpl::WriteImpl(const WriteOptions& write_options,
                         WriteBatch* my_batch, WriteCallback* callback,
                         uint64_t* log_used, uint64_t log_ref,
                         bool disable_memtable, uint64_t* seq_used) {
  ......
  if (immutable_db_options_.enable_pipelined_write) {
    return PipelinedWriteImpl(write_options, my_batch, callback, log_used,
                              log_ref, disable_memtable, seq_used);
  }
}
```

如果打开了enable_pipelined_write，则调用PipelinedWriteImpl

```cpp
Status DBImpl::PipelinedWriteImpl(const WriteOptions& write_options,
                               WriteBatch* my_batch, WriteCallback* callback,
                               uint64_t* log_used, uint64_t log_ref,
                               bool disable_memtable, uint64_t* seq_used) {
  ......
  // 1. 入队列newest_writer(不是leader则阻塞等待被唤醒)
  write_thread_.JoinBatchGroup(&w);
  if (w.state == WriteThread::STATE_GROUP_LEADER) {
    ......
    // 2. 如果自己是newest_writer_的leader，打包队列中的
    //    Writer到wal_write_group
    last_batch_group_size_ =
        write_thread_.EnterAsBatchGroupLeader(&w, &wal_write_group);
    // 3. 写WAL
    ......
    // 4. 将wal_write_group连到newest_memtable_writer_队列中，
    //    然后选择newest_writer_队列中新的leader并唤醒它
    //    (不是newest_memtable_writer_的leader则阻塞等待被唤醒)
    write_thread_.ExitAsBatchGroupLeader(wal_write_group, w.status);
  }
  if (w.state == WriteThread::STATE_MEMTABLE_WRITER_LEADER) {
    ......
    // 5. 如果自己是newest_memtable_writer_的leader，打包队列中的
    //    Writer到memtable_write_group
    write_thread_.EnterAsMemTableWriter(&w, &memtable_write_group);
    // 6. 写Memtable
    ......
    // 7. 选择newest_memtable_writer_队列中的新leader并唤醒它，
    //    然后更新memtable_write_group中所有writer的state并
    //    唤醒所有followers
    write_thread_.ExitAsMemTableWriter(&w, memtable_write_group);
  }
}
```

这就是pipeline的关键逻辑实现。两个队列的followers分别会阻塞在步骤1和4上，分别等待newest\_writer\_和newest\_memtable\_writer\_的leader来唤醒。

还有比较关键的两个地方就是ExitAsBatchGroupLeader和ExitAsMemtableLeader，涉及新leader的选择，两个队列的拼接，followers的唤醒等等，具体实现就不展开了，看代码很快。



## 总结

提高性能的一个重要手段就是pipeline，以后自己的项目中对于关键逻辑部分也要细化拆分做到尽可能的优。

**BUT:** 对于rocksdb的这个pipeline的优化，本人持怀疑态度，毕竟写WAL要比Memtable慢很多，可能每次从newest\_writer\_向newest\_memtable\_writer\_传batch的时候，后者基本都是空的。这样反而额外耗了CPU，得不偿失呀。这个pipeline的优化我还没有实测，这两天找时间测测，看看具体效果如何。