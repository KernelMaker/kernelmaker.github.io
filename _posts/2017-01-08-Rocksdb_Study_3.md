---
layout: post
title: 【Rocksdb实现分析及优化】 allow_concurrent_memtable_write
---

前面的写[博客](http://kernelmaker.github.io/Rocksdb_Study_1)总结过Rocksdb在JoinBatchGroup那里做的很细致的优化，来减少Writer在AwaitState的消耗。今天刚好有时间，所以测了一下这快优化的效果，刚好看到它有一个allow_concurrent_memtable_write的配置项，开启后可以并发写memtable，所以这篇先介绍下这个配置项原理及实现，再给出测试结果

## allow_concurrent_memtable_write

从名字就可以看出来这个配置项打开（5.0.1之前默认是关闭）之后可以并发写memtable，我们知道rocksdb是single writer的实现，之前也说过writebatch的作用，和leveldb一样所有的writer进行排队，每次只有队头的writer可写，其他writer等待，队头writer将自己和后面的一些writer的writebatch合并成一个大的writebatch并且执行，执行完之后标注好自己及它已经代执行过的writer的writebatch状态，唤醒那些已经代替其执行过的writer及新指定的下一个队头，开始下一波写，被唤醒的writer发现自己的writebatch状态时COMPELTE，则直接返回结果，如果是LEADER，则作为新的队头开始写剩余的writebatch。

这可以看到虽然队头writer会代其后的writer来执行他们的writebatch，但毕竟这些所有的任务还是有一个线程来做，有没有可能在这里并起来，有，这就是allow_concurrent_memtable_write的作用。

还是刚才说的过程，队头writer打包后面writer的writebatch，整体写完wal，此时，如果allow_concurrent_memtable_write打开，则依次唤醒这些writer让他们自己执行自己的writebatch，执行完之后继续AwaitState，等待最后一个执行完的writer将他们全部标记成COMPLETE并唤醒，完成，返回结果。

*注: 看代码里面WriteImpl()有个exit_completed_early，开始有点绕，原来这个就是说假如leader不是最后一个执行完的writer，此时被唤醒之后需不需要做收尾工作（执行ExitAsBatchGroupLeader），如果是false就由它来收尾，注意如果配置的是need_log_sync，则此时一定是先sync wal再ExitAsBatchGroupLeader唤醒其他writer，此时其他的写请求才能继续被执行*

以上就是allow_concurrent_memtable_write的原理，可以看到打开后memtable可以并发写提高性能，BUT，虽然写memtable可以并发，但所有的writer写完后还要进行AwaitState来等待汇总，这里额外的上下文切换会有额外的开销，搞不好并行写memtable带来的优化还cover不住这里的额外开销，对了，前面博客说过rocksdb对AwaitState有优化，只要打开enable_write_thread_adaptive_yield就可以使AwaitState在condition wait之前先yield几次，尽可能不去condition wait，嗯，开测

## 测试

### 1. 测试方法

1000万条不重复记录，每条记录如下

```
key: [0-10000000]__A_LONG_SUFFIX
value: 123456789012345678901234567890
```

是个线程，每个线程写100万条记录，分别测试三种配置下的用时：

1. 配置1：默认

   ```
   enable_write_thread_adaptive_yield = false
   allow_concurrent_memtable_write = false
   ```


2. 配置2：只打开enable_write_thread_adaptive_yield

   ```
   enable_write_thread_adaptive_yield = true
   allow_concurrent_memtable_write = false
   ```

3. 配置3：只打开allow_concurrent_memtable_write

   ```
   enable_write_thread_adaptive_yield = false
   allow_concurrent_memtable_write = true
   ```

4. 配置4：都打开

   ```
   enable_write_thread_adaptive_yield = true
   allow_concurrent_memtable_write = true
   ```

   ​

### 2. 测试结果

* 配置1：用时39s，cpu ~ 300%
* 配置2：用时33s，cpu ~ 700%
* 配置3：用时43s，cpu ~ 300%
* 配置4：用时29s，cpu ~ 700%



### 3. 结果分析

以默认配置（配置1）为标杆，可以看到

1. 配置2：如果仅打开enable_write_thread_adaptive_yield开启rocksdb的AwaitState优化，用时少10s，cpu耗用更多，起到优化效果（用cpu高耗用换高吞吐）
2. 配置3：如果仅打开allow_concurrent_memtable_write开启memtable并行写，性能不增反降，多了4s，这4s正是因为所有writer最后汇总而进行额外的AwaitState带来的开销的，cpu和默认一样
3. 配置4：如果都打开，性能最好，两个参数互相配合，用时减少了10s，cpu耗用高



## 总结

AwaitState的优化的确起到不少效果，配合并发写memtable，性能很高，如果再用上ColumnFamily，多个memtable的并发应该更好，不过这是cpu换吞吐，在真正使用之前还需要结合实际机器情况来权衡考虑



### 附测试代码

```c++
#include <string>
#include <thread>
#include <iostream>
#include <chrono>

#include "rocksdb/db.h"
#include "rocksdb/slice.h"
#include "rocksdb/options.h"

using namespace rocksdb;

const int kThreadNum = 10;
const long kNum = 10000000;
const std::string key_suffix = "_A_LONG_SUFFIX";
const std::string value = "123456789012345678901234567890";
//const std::string value = std::string(1024, 'a');

void Func(DB* db, long start, long count) {
  Status s;
  std::cout << "Thread " << std::this_thread::get_id() << " handle key range: "
            << start << " to " << start + count << std::endl;
  for (long i = 0; i < count; i++) {
    s = db->Put(WriteOptions(), std::to_string(i+start) + key_suffix, value);
    if (!s.ok()) {
      std::cout << "Thread " << std::this_thread::get_id()
                << " put error: " << s.ToString() << std::endl;
      break;
    }
  }
}

std::string kDBPath = "./testdb";

int main(int argc, char **argv) {

  DB* db;
  Options options;
  // create the DB if it's not already present
  options.create_if_missing = true;
//  options.write_buffer_size = 1024*1024*512;

  if (argc > 1) {
    std::cout << "use parallel" << std::endl;
    options.allow_concurrent_memtable_write = true;
    options.enable_write_thread_adaptive_yield = true;
  }

  std::cout << value << std::endl;

  // open DB
  Status s = DB::Open(options, kDBPath, &db);
  assert(s.ok());

  std::thread *threads[kThreadNum];
  auto start = std::chrono::steady_clock::now();
  for (int i = 0; i < kThreadNum; i++) {
    threads[i] = new std::thread(Func, 
                 db, i * kNum / kThreadNum, kNum / kThreadNum);
    std::this_thread::sleep_for(std::chrono::microseconds(100));
  }

  for (auto &t : threads) {
    t->join();
  }
  auto end = std::chrono::steady_clock::now();
  
  std::cout << "used " << (end-start).count() << std::endl;

  for (int i = 0; i < kThreadNum; i++) {
    delete threads[i];
  }

  delete db;

  return 0;
}
```

