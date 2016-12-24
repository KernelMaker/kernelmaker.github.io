---
layout: post
title: Leveldb之Put实现
---

这一篇主要总结一下leveldb put接口的实现，put接口用来向leveldb写入数据，leveldb的写入速度很快，主要因为它将随机写转化为顺序写，一个写入操作会首先在log文件中顺序写oplog，然后在内存的memtable中插入该数据就可返回，它的整体实现不难，逻辑也很清晰，这里做一下


## 调用栈

主要调用顺序大体就3步：
	
	DBImpl::Put -> DB::Put -> DBImpl::Write

下面来分别看一下

## DBImpl::Put

```cpp
Status DBImpl::Put(const WriteOptions& o, const Slice& key, const Slice& val) {
  return DB::Put(o, key, val);
}
```
这个接口是暴露给用户的接口，三个参数中后两个很明显，是key和value，第一个WriteOptions仅包含一个sync的bool成员用来告诉leveldb对于log是立刻刷盘还是交给操作系统，默认是false，下面在具体用到的地方会讲，可以看到这个接口什么也没做，仅仅是调用了DB::Put

## DB::Put

```cpp
Status DB::Put(const WriteOptions& opt, const Slice& key, const Slice& value) {
  WriteBatch batch;
  batch.Put(key, value);
  return Write(opt, &batch);
}
```
代码也很少，前两行构造出一个WriteBatch并且将key，value写入到WriteBatch中，然后当做参数调用DBImpl::Write，说一下WriteBatch，leveldb支持原子操作，就是可以将多个写操作放在一个WriteBatch中打包，然后在执行的过程中，WriteBatch中所有的写操作打包成一条日志存在log文件中，这时候即使在依次往memtable里插入时机器挂了也没事，下次重启的时候recover会将log中这个Writebatch重做，这样原子性就得到了保证

这里有一点想说明一下，leveldb没有对写memtable的结果进行判断，默认永远成功，通常写memtable失败的原因是在new一个skiplist的节点失败，leveldb在这里没有判断new的返回值默认永远成功，所以说这里能否保证原子性依赖于new失败之后在对其返回地址写的时候是不是100%崩溃，只要崩溃了，原子性还是有保证的，这个问题还需要仔细跟下去，这里是从Arena来取内存，我比较了一下，所有的new的左值都有可能是上一次的值，是脏值，是脏值的意思就是可能对它写不会造成程序立马崩溃，假如没有立马崩溃那岂不是原子性无法保证了？
	
WriteBatch的实现也不难，前8个字节是一个sequence number（Memtable的每次写入都会带一个递增的序列号，用作实现snapshot和区分同一个key多次复写的先后顺序，WriteBatch里的所有写入带的是同样的序列号），紧跟着4个字节是count代表这个WriteBatch中打包了几条操作，然后便依次是每条操作的记录(Type, key, [value])，这个WriteBatch构造好之后，就传给DBImpl::Write函数来做真正的写入

## DBImpl:Write

这个是真正的写入实现，我们分段来看，刚一开始有这样的操作：

```cpp
Writer w(&mutex_);
w.batch = my_batch;
w.sync = options.sync;
w.done = false;
```
Writer是每次写入过程的一个抽象，先来看定义：

```cpp
struct DBImpl::Writer {
  Status status; // 本次写入结果
  WriteBatch* batch; //本次写入的WriteBatch
  bool sync; //本次写入对应的log是否立刻刷盘
  bool done; //本次写入是否完成
  port::CondVar cv; //条件锁，如果前面有别的写入正在执行，本次写入
  //会Wait，等待正在执行的写入执行完后唤醒它

  explicit Writer(port::Mutex* mu) : cv(mu) { }
};
```
Writer生成好之后，接着是：

```cpp
MutexLock l(&mutex_);
writers_.push_back(&w);
while (!w.done && &w != writers_.front()) {
  w.cv.Wait();
}
if (w.done) {
  return w.status;
}
```
这段是比较精彩的地方，上来先抢一个mutex锁，可以看到多线程的写入会在这里互斥，只能有一个进来，提前剧透一下，第一个进来的写操作会在真正写log文件和写memtable的时候把锁放掉，这时候别的写操作会进来把自己push到writes_的deque中，然后睡在自己的条件锁上，为什么呢？因为此时自己的done不是true并且自己不是deque中的第一个writer，等第一个writer全部操作完之后呢，会把此时deque中的所有writer唤醒,这里可能要问了，为什么醒了之后第一步是判断自己是不是done呢，自己还没执行怎么可能会done呢，其实这就是leveldb的优化，当前抢到锁进来的writer会在真正操作之前把此时deque中所有的writer的任务都拿过来（实际上不是全拿过来，有数量限制），然后帮他们都做了，并且把结果放回至每个writer对应的status中，最后再唤醒所有writer，所以这些writer醒来后发现自己的任务已经被做了，就直接拿自己的返回值返回了，其实这就是后面部分的主要逻辑了，接着看代码：

```cpp
// May temporarily unlock and wait.
  Status status = MakeRoomForWrite(my_batch == NULL); // 做写入前的各种
  //检查，如是不是该停写，是不是该切memtable，是不是该compact
  uint64_t last_sequence = versions_->LastSequence(); // 取当前最大序列号
  Writer* last_writer = &w; // laster_writer的意思是打包多个write中的最
  //后一个writer
  if (status.ok() && my_batch != NULL) {  // NULL batch is for compactions
    WriteBatch* updates = BuildBatchGroup(&last_writer); // 合并writers里
    //多个writer的WriteBatch，就是上面说的deque里此时阻塞住的writer完成他
    //们的任务，打包后last_writer为该打包中的最后一个writer
    WriteBatchInternal::SetSequence(updates, last_sequence + 1); //本次写
    //入对应的序列号
    last_sequence += WriteBatchInternal::Count(updates); //更新last_sequence，
    //加的个数等于此次是WriteBatch中操作的个数

    // Add to log and apply to memtable.  We can release the lock
    // during this phase since &w is currently responsible for logging
    // and protects against concurrent loggers and concurrent writes
    // into mem_.
    {
      mutex_.Unlock(); //写日志和写memtable时可以放锁，
      //让别的write进入deque，尽可能减少互斥时间
      status = log_->AddRecord(WriteBatchInternal::Contents(updates)); //写日志
      bool sync_error = false;
      if (status.ok() && options.sync) {
        status = logfile_->Sync(); // 这里就是上面说的sync选项，
        //如果为true的话，对日志立刻刷盘
        if (!status.ok()) {
          sync_error = true;
        }
      }
      if (status.ok()) {
        status = WriteBatchInternal::InsertInto(updates, mem_); //将WriteBatch
        //中的操作依次写入memtable中
      }
      mutex_.Lock(); //再次加锁，需要互斥的更新versions的last_sequence
      if (sync_error) {
        // The state of the log file is indeterminate: the log record we
        // just added may or may not show up when the DB is re-opened.
        // So we force the DB into a mode where all future writes fail.
        RecordBackgroundError(status);
      }
    }
    if (updates == tmp_batch_) tmp_batch_->Clear();

    versions_->SetLastSequence(last_sequence);
  }
  //从deque的第一个writer开始pop，第一个writer一定是当前操作的writer，
  //直到pop到本次打包的最后一个writer
  while (true) {
    Writer* ready = writers_.front();
    writers_.pop_front();
    if (ready != &w) {
      ready->status = status; //设置返回结果
      ready->done = true; 
      ready->cv.Signal(); //唤醒
    }
    if (ready == last_writer) break;
  }

  // Notify new head of write queue
  //唤醒此时的deque头writer，从它开始进行接下来的写入
  if (!writers_.empty()) {
    writers_.front()->cv.Signal();
  }

  return status;
```

关键部分都已经写在注释里，其实就是上面说的过程，很清晰，这里有一个函数需要特殊说明一下：

```cpp
Status DBImpl::MakeRoomForWrite(bool force) {
  mutex_.AssertHeld(); //此次是占有锁的
  assert(!writers_.empty());
  bool allow_delay = !force;
  Status s;
  while (true) {
    if (!bg_error_.ok()) { //如果此时已经发生了错误，直接返回错误
      // Yield previous error
      s = bg_error_;
      break;
    } else if (
        allow_delay &&
        versions_->NumLevelFiles(0) >= config::kL0_SlowdownWritesTrigger) {
      // We are getting close to hitting a hard limit on the number of
      // L0 files.  Rather than delaying a single write by several
      // seconds when we hit the hard limit, start delaying each
      // individual write by 1ms to reduce latency variance.  Also,
      // this delay hands over some CPU to the compaction thread in
      // case it is sharing the same core as the writer.
      //如果此时level 0中的文件个数已经大于等于kL0_SlowdownWritesTrigger，
      //放锁，sleep一秒来降低写入速度，给level 0的compact留出时间
      mutex_.Unlock();
      env_->SleepForMicroseconds(1000);
      allow_delay = false;  // Do not delay a single write more than once
      mutex_.Lock();
    } else if (!force &&
               (mem_->ApproximateMemoryUsage() <= options_.write_buffer_size)) {
      //如果此时memtable还有剩余空间，则不需要申请新的memtable，直接返回就好了
      // There is room in current memtable
      break;
    } else if (imm_ != NULL) {
      // 如果此时immutable memtable不为空，代表正在flush，此时memtable也满了，
      // 只能停写等待flush结束，结束后会被唤醒，注意这里不放锁
      // We have filled up the current memtable, but the previous
      // one is still being compacted, so we wait.
      Log(options_.info_log, "Current memtable full; waiting...\n");
      bg_cv_.Wait();
    } else if (versions_->NumLevelFiles(0) >= config::kL0_StopWritesTrigger) {
      //如果此时level 0的文件个数大于等于config::kL0_StopWritesTrigger，停写，
      //注意，这里不放锁，等compact结束后被唤醒
      // There are too many level-0 files.
      Log(options_.info_log, "Too many L0 files; waiting...\n");
      bg_cv_.Wait();
    } else {
      // memtable已经满了并且此时没有immutable memtable，切memtable成imm并生成
      //新的memtable，切日志
      // Attempt to switch to a new memtable and trigger compaction of old
      assert(versions_->PrevLogNumber() == 0);
      uint64_t new_log_number = versions_->NewFileNumber();
      WritableFile* lfile = NULL;
      s = env_->NewWritableFile(LogFileName(dbname_, new_log_number), &lfile);
      if (!s.ok()) {
        // Avoid chewing through file number space in a tight loop.
        versions_->ReuseFileNumber(new_log_number);
        break;
      }
      delete log_;
      delete logfile_;
      logfile_ = lfile;
      logfile_number_ = new_log_number;
      log_ = new log::Writer(lfile);
      imm_ = mem_;
      has_imm_.Release_Store(imm_);
      mem_ = new MemTable(internal_comparator_); // 生成新的memtable
      mem_->Ref();
      force = false;   // Do not force another compaction if have room
      MaybeScheduleCompaction(); //检查是否需要做compact，需要的话给后台
      //任务线程发送compact任务
    }
  }
  return s;
```
可以看到，这个函数虽然名字上感觉是检查memtable的可用大小，如果不够就切新的memtable，实际上做的检查还是很多的，这些检查主要都是因为写入压力过大速度太快而造成的，如memtable已经满了，但此时的immutable memtable还没flush完，需要等，或者此时level 0的文件数已经很多了，需要减慢写或者停写，否则的话因为level 0的文件keyrange会有覆盖，本层的读是需要读每个sst文件，会造成读放大，所以不能放任其文件数增长而不管，必要时需要停写来等level 0 的文件都compact到level 1

## 总结
leveldb的写入实现差不多就是这样，总体来说不难，实现的也非常的好，还是很有学习的必要
