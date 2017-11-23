---
layout: post
title: 【Rocksdb实现分析及优化】Write Ahead Log刷盘策略及实现
---

rocksdb在写memtable之前，会先写WAL，所以WAL的刷盘策略很重要，事关机器宕机后数据是否丢失的问题，看了下最新的v5.8版本的代码，这里简单总结下这里吧



## 1. 相关配置

options中和WAL刷盘策略相关的配置只有一个：

```
uint64_t wal_bytes_per_sync = 0;
```

怎么用后面讲，另外注意direct_io对WAL是不生效的。



## 2. WAL文件操作封装

WAL操作类从上到下的封装如下：

```cpp
log::Write

WritableFileWriter

PosixWritableFile

fd
```

另外，在创建WAL之前，首先会有一个调用如下

```cpp
  EnvOptions OptimizeForLogWrite(const EnvOptions& env_options,
                                 const DBOptions& db_options) 
                                 const override {
    EnvOptions optimized = env_options;
    optimized.use_mmap_writes = false;
    optimized.use_direct_writes = false;
    optimized.bytes_per_sync = db_options.wal_bytes_per_sync;
    optimized.fallocate_with_keep_size = true;
    return optimized;
  }
```

可以看到，这里会不用direct_io和mmap打开文件，然后将用户配置的wal_bytes_per_sync赋值给EnvOptions，然后调用

```cpp
s = NewWritableFile(
            env_, LogFileName(immutable_db_options_.wal_dir, new_log_number),
            &lfile, opt_env_opt);
```

使用上面optimized EnvOptions打开PosixWritableFile(即lfile)，它里面会真正open一个fd。接着使用lfile构造一个WritableFileWrite对象file_writer，最后使用file_writer构造最终的log::Writer对象。

这里有个细节，上面的optimized.fallocate_with_keep_size = true是干什么用的呢，等会说。在构造好lfile后，会根据write_buffer_manager、db_write_buffer_size和max_total_wal_size的配置大小算出一个合理值，将它赋值给lfile的preallocation_block_size_。这个又是干什么用的呢，原来在调用file_writer->Append时，会调用lfile->PrepareWrite，它里面会根据preallocation_block_size\_的算出offset和len，传入Allocate，Allocate实现如下：

```cpp
#ifdef ROCKSDB_FALLOCATE_PRESENT
Status PosixWritableFile::Allocate(uint64_t offset, uint64_t len) {
  assert(offset <= std::numeric_limits<off_t>::max());
  assert(len <= std::numeric_limits<off_t>::max());
  TEST_KILL_RANDOM("PosixWritableFile::Allocate:0", rocksdb_kill_odds);
  IOSTATS_TIMER_GUARD(allocate_nanos);
  int alloc_status = 0;
  if (allow_fallocate_) {
    alloc_status = fallocate(
        fd_, fallocate_with_keep_size_ ? FALLOC_FL_KEEP_SIZE : 0,
        static_cast<off_t>(offset), static_cast<off_t>(len));
  }
  if (alloc_status == 0) {
    return Status::OK();
  } else {
    return IOError(
        "While fallocate offset " + ToString(offset) + " len " +
         ToString(len),
        filename_, errno);
  }
}
#endif
```

man 2一下fallocate

> fallocate() allows the caller to directly manipulate the allocated disk space
>
> for the file referred to by fd for the byte range starting at offset and
>
> continuing for len bytes.
>
> 
>
> FALLOC_FL_KEEP_SIZE
>
> This  flag allocates and initializes to zero the disk
> space within the range specified by offset and len. 
> After a successful call, subsequent writes into this
> range are guaranteed not to fail because
> of lack of disk space.  Preallocating zeroed blocks
> beyond the end of the file is useful for optimizing 
> append workloads.  
> ​              
> Preallocating blocks does not change the file size
> (as reported  by  stat(2))
> even if it is less than offset+len.

这里就明了了，其实就给预分配空间（比如分配write_buffer_size大小的空间），确保对fd的写入不会因为磁盘空间不足而失败，所以刚才提问的optimized.fallocate_with_keep_size = true其实就是此处的FALLOC_FL_KEEP_SIZE，有了它，fallocate即使offset大于当前文件大小，也不会改变文件大小。



扯了这么多，其实就是缕了一下rocksdb到底对WAL最低层的fd怎么封装的已经相关配置到底怎么用，个人感觉封装的有点太多，缕起来有点麻烦，不过我这种细节狂魔还是想一探究竟，强迫症。。。



## 2. 三种策略

#### 1. 每条都刷盘

如果对数据安全性要求特别高，可以在Put或者Write是，配置WriteOptions::sync = true，这样在写完日志后会立刻刷盘，实现如下：

```cpp
Status DBImpl::WriteToWAL(const WriteThread::WriteGroup& write_group,
                          log::Writer* log_writer, uint64_t* log_used,
                          bool need_log_sync, bool need_log_dir_sync,
                          SequenceNumber sequence) {
  ......
  
  if (status.ok() && need_log_sync) {
    StopWatch sw(env_, stats_, WAL_FILE_SYNC_MICROS);
    // It's safe to access logs_ with unlocked mutex_ here because:
    //  - we've set getting_synced=true for all logs,
    //    so other threads won't pop from logs_ while we're here,
    //  - only writer thread can push to logs_, and we're in
    //    writer thread, so no one will push to logs_,
    //  - as long as other threads don't modify it, it's safe to read
    //    from std::deque from multiple threads concurrently.
    for (auto& log : logs_) {
      status = log.writer->file()->Sync(immutable_db_options_.use_fsync);
      if (!status.ok()) {
        break;
      }
    }
    if (status.ok() && need_log_dir_sync) {
      // We only sync WAL directory the first time WAL syncing is
      // requested, so that in case users never turn on WAL sync,
      // we can avoid the disk I/O in the write code path.
      status = directories_.GetWalDir()->Fsync();
    }
  }
  ......
}
```

可以看到如果配置了sync=true，则need_log_sync=true，然后会执行WritableFileWriter::Sync，先不管里面细节，总之这里面最终会执行fsync确保刷盘。

所以这种方式是最安全的，只要写入成功即使立刻宕机，数据也不会丢，不过性能最差



#### 2. 配置了wal_bytes_per_sync 

如果配置了wal_bytes_per_sync不为0，假如1M，则WAL按照写入顺序，没写入1M就将其前面1M的内容刷盘，可以理解成1M 1M的刷，这样相对每一条都sync的策略来说，sync频率变低，性能会高，但会有丢数据的风险，最多丢wal_bytes_per_sync字节

wal_bytes_per_sync到底在底下怎么做的呢，有必要看一下。

它会在构造WritableFileWriter对象（即上面说的file_writer）时传入构造函数，赋值给lfile的bytes_per_sync_成员变量，然后在file_writer->Append中，并不是直接写文件，而是放到其缓冲区buf\_中，然后调用file_writer->Flush，在Flush里面才会对其成员PosixWritableFile（即上面说的lfile）调用lfile->Append，真正写到PageCache中，至于这里为什么file\_writer->Append会先把数据放到buf\_中，我觉的应该是如果配置了rate_limiter，把数据缓存在内存里然后按照限速策略来一点一点写，最终达到一个限速的目的吧。

接着来，写如PageCache后，file_writer->Flush还会做如下逻辑：

```cpp
Status WritableFileWriter::Flush() {
  ......
  
  if (!use_direct_io() && bytes_per_sync_) {
    const uint64_t kBytesNotSyncRange = 1024 * 1024;  // recent 1MB
                                                      // is not synced.
    const uint64_t kBytesAlignWhenSync = 4 * 1024;    // Align 4KB.
    if (filesize_ > kBytesNotSyncRange) {
      uint64_t offset_sync_to = filesize_ - kBytesNotSyncRange;
      offset_sync_to -= offset_sync_to % kBytesAlignWhenSync;
      assert(offset_sync_to >= last_sync_size_);
      if (offset_sync_to > 0 &&
          offset_sync_to - last_sync_size_ >= bytes_per_sync_) {
        s = RangeSync(last_sync_size_, offset_sync_to - last_sync_size_);
        last_sync_size_ = offset_sync_to;
      }
    }
  }
  
  return s;
}
```

逻辑就是如果距上次sync的位置last_sync_size_，文件新的大小filesize\_ 减去kBytesNotSyncRange（意思是最新写入的1M数据不做sync）之后的大小如果大于bytes_per_sync\_，则对这部分数据进行RangeSync，RangeSync最终调用sync_file_range来完成这部分数据的sync，调用之后，除了文件最后的1M数据之外，其他的内容都已经刷盘成功。



#### 3. 完全交给操作系统

这种性能最高，但一旦宕机，丢失的数据相对前面两种策略也是最多的。rocksdb默认用的是这个策略。

有一点需要注意，这种策略下，rocksdb并不是完全不主动sync，当有多个columnfamily并且需要切换WAL文件时（Flush memtable前），rocksdb会强制把之前的WAL都刷盘，否则之前对多个columnfamily写入成功的batch操作会丢失部分数据变的不一致。举个例子：

假如有A、B两个columnfamily，他们共享一个WAL，某一时刻A需要Flush memtable，此时它会切换到新的WAL，并且将memtable里的内容写到Level 0的sst文件并且刷盘，注意，sst文件刷盘了！假设在A Flush之前有一个batch操作分别对A、B写入一条数据，Flush之后对A写入的数据已经存在于sst中并刷盘，不会丢，而对B写入的数据还在B的memtable以及上一个WAL中，如果此时不对WAL刷盘，发生宕机并重启，上一个WAL对B写入的那条数据记录丢失，不能recover，这时候就发生batch不一致了。



## 总结

清楚rocksdb WAL的刷盘策略还是很有必要的，这样才能根据自己数据的重要程度来选择合适的策略