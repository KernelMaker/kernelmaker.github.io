---
layout: post
title: pika_hub debug日记2
---



一个月前修复了pika_hub丢数据的问题，以为没有问题了（反复测试5遍都通过），单机版本pika_hub足够稳定，所以十一后开始捣鼓多机pika_hub，谁料刚写了个开头，居然又发现丢数据的问题T_T。又是痛苦好一阵子的找BUG过程，终于，今天揪出这货。写篇博客记录一下，方便自己今后回顾。

## 1. 现象

3个pika实例先各灌1000万的数据，然后再启动pika_hub开始进行同步，通过info命令确定同步完成，并且日志没有报任何错误，这种情况下，低概率的出现部分pika实例缺失百十来条的数据。

在pika_hub的binlog文件中定位，这些丢失的数据基本连续在一起，差不多在一个block中，然后这个block在binlog文件的位置不固定。



## 2. 排查过程

1. 开始以为是升级floyd和pink，他们里面的网络处理的BUG导致的，回滚，还是会复现...
2. 然后确定是节后加多机pika_hub逻辑带来的bug，一路回滚测试，继续复现...
3. 回滚到节前单机稳定版本，排查是否这个问题早就存在，终于在连续测试第8次的时候复现...
4. 好吧，到这里，可以猜测问题是出在binlog边读边写的造成的，重新缕了一下rocksutil的LogReader和LogWriter，加了各种调试信息，开始疯狂测试...
5. 终于，在连续测试几十次的时候，复现了完美的现场，根据调试信息，找到了该死的BUG！



## 3. 原因

调试信息打出如下内容：

```
ReadPhysicalRecord kBadHeader
kBadHeader
ReadPhysicalRecord kBadRecord
kBadRecord
```

位置连续，那么就在rocksutil的log_reader文件中找答案。

问题出现的合理过程推测如下：

1. 某次ReadPhysicalRecord，判断buffer\_.size() < (size_t)kHeaderSize，进行ReadMore
2. 当前不是eof\_和read\_error\_，进行Read，然后buffer\_.size() < (size_t)kBlockSize，标记eof_ = true
3. 返回ReadPhysicalRecord，解析buffer\_的内容，发现header_size + length > buffer\_.size()，**清空buffer\_**，当前eof_==true并且drop_size不为空，所以返回kBadHeader
4. 返回到ReadRecord中，case kBadHeader，因为WALRecoveryMode是默认值，不等于kAbsoluteConsistency，所以这里**没有ReportCorruption**，然后继续while循环调用ReadPhysicalRecord
5. 进入ReadPhysicalRecord中，因为buffer\_之前被清空，需要继续ReadMore
6. 进入ReadMore，因为eof\_==true，进入else分支，此时由于buffer\_是空，不会进入if，故将error标记成kEof，return false
7. 一路返回到pika_hub调用ReadRecord的地方，返回false，但是report没有报告错误，所以认为是EOF，pika_hub的这个BinlogReader进入wait状态
8. 当新数据写入，BinlogWriter唤醒了这个BinlogReader，BinlogReader执行UnmarkEOF
9. 在UnmarkEOF中，刚好读满了一个Block，将eof\_置成false，注意，因为这个block前半部分之前被清空，这里读出来的后半部分最终会解析失败的
10. UnmarkEOF之后，BinlogReader继续调用ReadRecord，ReadRecord继续调用ReadPhysicalRecord，当前buffer\_大于kHeaderSize，所以不用ReadMore，直接往下走，开始解析，此时巧了，解析出来的type=kZeroType并且length=0，**再次清空buffer_，此时这个出问题的Block已经整体被清空了**，然后返回kBadRecord
11. 返回到ReadRecord中，case kBadRecord，同样WALRecoveryMode是默认值，不等于kAbsoluteConsistency，所以这里**没有ReportCorruption**，然后继续while循环调用ReadPhysicalRecord，开始读下一个Block

至此，由于边读边写加上WALRecoverMode不是kAbsoluteConsistency，这个Block在没有返回任何错误的情况下被略过，默认32K的Block，会丢失不少数据… … WTF



## 4. 解决办法

WALRecoverMode设置成kAbsoluteConsistency，这样ReadRecord的错误可以比较严格的被Report，这样BinlogReader在发现ReadRecord return false的时候，会拿到这个错误，就不会错当成EOF停下来了

现在不也能确定这样可以彻底解决，因为有一些ReportCorruption的地方还有条件的，要是恰巧没进去可能还是有问题，另外还发现过BinlogReader一直在kOldRecord循环出不来的现象，不过原因就在这里了，如果将来还有问题，把ReadRecord的case这里逻辑加强一下，确保都ReportCorruption和break。



## 5. 意外发现

因为加了很多调试信息，意外发现两个问题：

1. BinlogWriter会把收到的命令先过滤一下LRU，如果全被过滤掉了，AddRecord有可能写一个Header+空信息进Binlog，然后BinlogReader会把它读出来，虽然正确性没问题，不过多此一举
2. 之前写的如果解析Binlog失败（如checksum）的话，会从记录的上一次成功读的位置开始重新创建一个新的BinlogReader，这样其实也会丢失信息，因为BinlogReader的offset是LogReader的end_of_buffer_offset\_，ReadMore是一个Block一个Block的读，假如这个Block读出来成功解析第一条，然后把end_of_buffer_offset\_当做它的offset记录下来，解析第二条失败，然后重新从end_of_buffer_offset\_创建新的BinlogReader，由于end_of_buffer_offset\_其实并不是第一条真正的offset，所以会略过很多还没有解析的信息，造成丢失。解决办法就是每次重建BinlogReader都从断点文件的最开始读，反正保证幂等，重复执行也无大碍。

## 6. 总结

既然上了自己造的“Binlog边写变读 + 用重试解决边读边写带来可能错误”的贼船，只能硬着继续开下去了，毕竟我觉得这样的方式还是比“安全”的读写互斥性能更高，但愿后面不要再有丢数据的问题了T_T，这样的BUG解决一个太伤了...