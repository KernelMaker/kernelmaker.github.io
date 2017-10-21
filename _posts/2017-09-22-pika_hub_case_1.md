---
layout: post
title: pika_hub debug日记1
---



最近同事测试pika_hub时发现在某些情况下多实例同步会丢数据，测试5个pika实例相互同步总量5000万条数据，结果有2实例各丢10条左右，找了好一阵子，最终解决了，写篇博客给自己记录一下吧

### 1. pika_hub如何读写binlog

在写pika_hub前，自己已经对pika现有的binlog读写方式不太满意，在多线程边读边写的场景，pika是这么做的：

```
1. 写binlog前加锁，这个没什么异议，为了保证写入的完整。但有一个
   manifest文件很蛋疼，这个就是为了记录写入的当前偏移量，其实用
   binlog文件的自然大小就可以的，可当初还是用了一个manifest来
   记录，导致每写一条就得更新manifest，还得sync一下，很丑很低效。
2. 读binlog前加锁，判断自己上次读的问题和binlog现在位置的大小，
   一样的话则放锁sleep一段时间重试，不一样的话，放锁读。这本身
   没什么问题，可以每次读之前都加锁判断一下，很影响writer的效率，
   同样低效
```

反思了pika的问题后，我在pika_hub的binlog实现中采用了新的做法

```
1. 直接使用文件的自然大小，去掉多余的manifest，并且每一条record
   都做了crc32校验
2. reader在读之前不需要加锁，直接无脑读，如果判断到EOF时再加锁，
   wait到condition上，writer后续写入后会signal，唤醒reader接着读
```



## 2. Bug定位

测试发现但凡丢的数据都是在个别binlog的最后，排查各种其他原因后，最终猜测应该是在binlog切文件时引发的问题。reader读binlog的过程大概如下：

```
while (!should_exit_) {
    ret = reader_->ReadRecord(&record, &scratch);
    if (ret) {
      DecodeBinlogContent(record, result);
      return rocksutil::Status::OK();
    } else {
      if (status_.ok()) {
        manager_->mutex()->Lock();
        manager_->GetWriterOffset(&writer_number, &writer_offset);
        reader_offset = reader_->EndOfBufferOffset();
        while (number_ == writer_number && reader_offset == writer_offset) {
          /*
           * if exit_at_end_ is TRUE,
           * return directly when all the content have been read
           * this mode is used in BinlogManager lru recovering
           */
          if (exit_at_end_) {
            manager_->mutex()->Unlock();
            return rocksutil::Status::Corruption("Exit");
          }
          /*
           * if exit_at_end_ is FALSE
           * wait until new content is written or should exit;
           */
          manager_->cv()->Wait();
          if (should_exit_) {
            manager_->mutex()->Unlock();
            return rocksutil::Status::Corruption("Exit");
          }
          manager_->GetWriterOffset(&writer_number, &writer_offset);
          reader_offset = reader_->EndOfBufferOffset();
        }
        
        reader_->UnmarkEOF();
        manager_->mutex()->Unlock();
        rocksutil::Status s = env_->FileExists(log_path_ + "/" + kBinlogPrefix +
              std::to_string(number_+1));
        if (s.ok()) {
          TryToRollFile();
        }
        } else {
        return status_;
      }
    }
  }
```

当被writer唤醒后，reader会检查writer当前的offset和自己的offset，如果一样，表示没有新内容写入，就继续wait，如果不一样（filenum不一样或者文件内offset不一样），则跳出循环，此时reader会判断下一个binlog文件是否存在，如果存在，就切到下一个文件去读，如果不存在，就继续读本文件。

问题就出在这里，假设reader在读binlog_1的末尾时发现EOF，进行wait，然后writer又在binlog\_1的末尾写了一条数据，并且唤醒reader，并且！此时writer判断binlog\_1大小大于100M需要切到binlog\_2，如果在reader判断是否存在下一个文件之前，writer就创建了binlog\_2，那么reader进行判断后就直接切到binlog\_2开始读，导致binlog\_1的最后几条数据丢失。

原因找到了，解决也很简单，在被唤醒并且跳出condition循环后，先判断当前文件实际大小和自己的read_offset是否一样，一样的话肯定是读完当前文件了，接着判断是否要切下一个文件读，不一样的话，就接着读当前文件，不切。



## 3. 意外发现

这个bug其实也不难（往往找到后都觉得不难^^），不过好玩的是在修复的时候，我发现了新的问题。之前在reader被唤醒之后，做的是先UnmarkEOF再Unlock，后来感觉没必要在锁里UnmarkEOF，就改成先Unlock再UnmarkEOF，然后测试就会报checksum错误，很好奇原因，就继续排查了一下。

先说下UnmarkEOF用来干嘛，rocksutil对wal文件的ReadRecord实现中，如果读到文件尾，就会进入eof状态，怎么判断是文件尾呢，就是某次Read出来的内容大小不足一个Block的大小（32K）。如果后续文件又写了新的内容，在接着读之前，需要执行UnmarkEOF，先将上次读取的最后一个不全的Block读完，这样后续的Read才能继续每次按照32K来卡。

既然UnmarkEOF就是做了次read，那么放在锁里和锁外为什么差别这么大呢？

放在锁里没问题，在UnmarkEOF进行read时，writer肯定不会写新的内容，放在锁外的话，read时writer有可能会在写新的内容。

我给rocksutil的log_reader加了各种调试日志，跑了N多次，最终找到了原因：

放在锁内，read时肯定读出的是完整的一条record（至少内容和Header记录相符，因为肯定是writer写完这条record后才唤醒reader），如果放在锁外，当UnmarkEOF进行read尝试补齐上次没读全的Block时，writer拿到锁会同时写这个Block剩余部分，所以reader有可能read出一条半的内容，当开始解析时，首先第一条没有问题，解析成功。剩下的半条，如果小于kHeaderSize，那么就会尝试ReadMore，也没问题，如果大于KHeaderSize，但通过它的Header拿到的record length大于当前buffer里的内容大小，rocksutil会直接drop掉这个buffer(因为rocksutil实现的是write ahead log，默认读写文件不是同时进行，读的时候出现这个问题，肯定是当初写文件写到最后没写完崩掉了，所以要丢弃这些垃圾内容)，如果这个buffer的内容被丢弃掉，返回kBadHeader，那么后续的读取全都发生错位，我看到的现象是后续拿到type=12（kOldRecord），然后不再读取新内容，解析停止了。

所以，UnmarkEOF只能放在锁里，不然read和write同时进行，很容易读出半条信息，而在rocksutil的实现里，半条信息的错误是不可恢复的。



## 3. 总结

pika_hub对rocksutil里write ahead log的用法还是有些风险，毕竟那本是write ahead log，不是专门用来边读边写的。理论上即使不在UnmarkEOF，正常的ReadRecord里也会读出半条信息，只不过实际线上读写不用那么巧合碰到一起。我暂时选择继续使用rocksutil的wal来做binlog，只不过在pika_hub里加了容错，一旦出现checksum错误，我会立刻关闭这个reader，重新打开新的reader，定位到上次的成功读取的位置，尝试再次读取这里，一般情况下，writer此时已经写完了这个block，会成功的。

等pika_hub写完之后，感觉有必要给rocksutil加一个真正适合边读边写的Binlog实现了。