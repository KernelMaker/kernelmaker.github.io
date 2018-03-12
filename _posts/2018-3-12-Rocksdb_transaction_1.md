---
layout: post
title: 【Rocksdb实现分析及优化】事务之Pessimistic ①
---

之前对Rocksdb都是关于它主体实现的介绍，最近在看他的事务实现，还没全看完，打算边看边记录，用几篇博客详细写一下Rocksdb Transaction相关的原理及实现。这一篇先不展开细节实现，简单介绍一下它的Pessimistic Transaction。

## 1. Pessimistic vs Optimistic

Rocksdb的事务实现包括Pessimistic（悲观）和Optimistic（乐观）。

Pessimistic transaction会在进行的过程中，当需要对某个key做Write操作时，首先对其加行锁，确保这个key在Commit或Abort之前不会被其他并行事务修改，然后在最终Commit或Abort后对所有行锁进行释放。当并发事务较高，并且事务间执行Write操作的key重叠度较高时，此时就需要采用Pessimistic transaction通过行锁来实现事务间的操作隔离。当然他的代价就是额外的锁操作。

Optimistic transaction则相反，Commit之前的所有Write操作都不需要加锁，冲突检测会在Commit时来执行，没有冲突则Commit成功，有冲突则Commit失败。当并发事务之间Write操作的key重叠度较低时，如果采用Pessimistic transaction反而会因为引入额外的锁操作反而降低性能，这里更适合使用Optimistic。Rocksdb默认是Pessimistic。

## 2. ACID

数据库事务需要满足ACID（Atomic、Consistency、Isolation、Durability）。这些正常应该是MyRocks或者InnoDB关心并实现的特性，而Rocksdb身为MyRocks的基础引擎，没有要求说一定得满足ACID，不过这里我们尝试强行套一下概念，来看看Rocksdb满足了哪些？

1. Atomic：原子性通过Rocksdb已有的WriteBatch就可以实现，事务在Commit之前所有的Write操作都只是写到WriteBatch当中，在最终Commit时只需要如下一个Write(...)即可：

```cpp
Status WriteCommittedTxn::CommitWithoutPrepareInternal() {
  Status s = db_->Write(write_options_, GetWriteBatch()->GetWriteBatch());
  return s;
}
```

2. Consistency：一致性似乎不需要Rocksdb做什么，应该是MyRocks需要解决的事情；
3. Isolation：隔离性通过行锁、Snapshot配合来实现，一会下面单独说；
4. Durability：持久性，这个比较尴尬，我翻看代码，并没有看到上面Commit时Write调用传入的WriteOptions::sync被打开，也就是WAL走的是默认的刷盘策略（前面的博客有介绍），在机器掉电时会有丢失的可能。这就需要使用者在BeginTransaction时，将传入的WriteOptions中的sync手动设置为true；

可以看到Rocksdb的Transaction已经基本实现了AID；

## 3. 怎么用

Pessimistic transaction怎么用，和普通的事物一样，一开始需要TransactionBegin，最后需要Commit或者RollBack。下面看几个例子:

1. 打开DB：

```cpp
Options options;
TransactionDBOptions txn_db_options;
options.create_if_missing = true;
TransactionDB* txn_db;

Status s = TransactionDB::Open(options, txn_db_options, kDBPath, &txn_db);
assert(s.ok());
```

2. 开启事务：

```cpp
Transaction* txn = txn_db->BeginTransaction(write_options);
assert(txn);
```

3. 读写：

```cpp
// 在事务中读取一个key
s = txn->Get(read_options, "abc", &value);
assert(s.IsNotFound());

// 在事务中写一个key
s = txn->Put("abc", "def");
assert(s.ok());

// 在事务外读取一个key
s = txn_db->Get(read_options, "abc", &value);

// 在事务外写一个key
// 这里并不会有影响，因为写的不是"abc"
// 如果是"abc"的话
// 则Put会一直卡住直到等待事务Commit或者超时(本例中会超时)
s = txn_db->Put(write_options, "xyz", "zzz");
```

4. 提交：

```cpp
s = txn->Commit();
assert(s.ok());
```

5. 析构事务：

```cpp
delete txn;
```

6. 最后析构DB：

```cpp
delete txn_db;
```



## 4. Isolation

一般数据库事务隔离级别从弱到强分为：

1. Read Uncommited：最弱的隔离级别，对读操作不加锁，对写加锁，会造成脏读
2. Read Commited：读的时候加读锁，读完立刻放锁，写的时候加写锁，事务结束后放锁。这样的隔离级别会有不可重复读的问题
3. Repeatable Reads：读的时候加读锁，事务结束后放锁，写的时候加写锁，事务结束后放锁。因为读锁在加锁后直到事务结束后才释放，所以其他事务无法修改对应key，这样就可以支持重复读，但还会存在**幻读(Phantom Reads)**问题
4. Serializable：这事最严格的隔离界别，读的时候对**表**加读锁，事务结束后放锁，写的时候对表加写锁，事务结束后放锁，由于都是对表加锁，所以没有幻读问题



知道了上述隔离级别的基本区别后，我们来看看Rocksdb的Pessimistic transaction在这方面是怎么做的。

1. PessimisticTransaction的Put 会在一开始对key加写锁，直到事务结束后才释放
2. PessimisticTransaction的Get 则不会对key加锁，它能读到什么数据完全取决于Options传入的Snapshot

> 开始看这里代码的时候，心里OS：“什么鬼？Get连锁都没有…”，后来发现还有一个GetForUpdate

3. PessimisticTransaction的GetForUpdate 则会对key加读锁，直到事务结束后才释放。这个才是在事务中应该使用的读取操作，配合Put的写锁来实现一定程度上的隔离性。同样它能读到什么值也要取决于Options传入的Snapshot

有了Put和GetForUpdate的加锁，我们对比上面列出的隔离界别实现原理，可以发现：

**Rocksdb Pessimistic Transaction的隔离界别是Repeatable Reads**（当然也不可能是Serializable，一是因为Rocksdb暴露的接口不会造成幻读，所以不需要一把大锁来强行Serialize，二是Rocksdb也没有表的概念）

我们发现，Rocksdb对于加锁的时机是在需要读写某个key之前才加锁，这样的方式可以满足绝大多数的事务隔离场景，当然在有些极端场景下，用户希望在事务一开始就对所有接下来需要读写key加锁，直到事务结束后再释放。这个该如何实现呢？

要实现这个，有一个需要解决的问题：

> 无法在事务一开始就知道接下来要读写哪些key，所以无法提前对它们加锁

Rocksdb通过SetSnapshot来间接解决，BeginTransaction后，用户可以立刻调用SetSnapshot，这样该事务会记录当前DB的最新Sequence，然后再接下来事务的每一次Put操作时，会检查DB中该key的最新Sequence是否大于事务之前记录的Sequence，如果大于，则证明事务外部有对该key做了写操作，那么事务对应的Put则会返回失败。

可以看到，在这里Rocksdb采用相对乐观的处理方式，通过对Snapshot来取代提前对所有需要的key加锁，不过这样的处理方式可能会造成事务最终失败（事务外部修改某个key后会导致接下来事务内部修改失败）。

前面总说事务外部读写，这里除了其他事务可能读写外，也有可能是用户直接使用DB接口来读写，也就是：TransactionDB::Put和TransactionDB::Get，具体实现中也是将它们当成一个独立的事务（只有一条操作Put或Get）来处理。

## 5. 其他

看代码中发现一些有趣的功能：

1. WriteBatchWithIndex：事务的Get和GetForUpdate操作可以读取到本事务中之前写过但还没有写入到DB中（未Commit）的最新值，具体实现就是通过WriteBatchWithIndex，它其实就是在WriteBatch基础之上加一个关于index的skiplist，这样就Get时可以在读Memtable之前先读在WriteBatch中查查看有没有最新写入。这个不止可以在Transaction中用，Rocksdb把它暴露了出来，如果在使用Rocksdb的时候有中途在WriteBatch里查找key的需求，就可以使用。
2. SetSavePoint & RollbackToSavePoint：在事务的进行中，可以随时调用SetSavePoint来打记录点，然后在接下来可以随时回滚到上次打的记录点上。具体实现就是在SetSavePoint是记录一下事务的状态信息（如snapshot、num_puts、num_deletes、num_merges）以及对WriteBatch设置记录点（Record个数，rep当前size），然后在Rollback时恢复他们即可。
3. 所有的锁操作默认不会无限卡死等待，会有超时时间，超时后将错误返回给用户来处理，这样更为合理。另外如果需要的话，可以开启事务Options的detect_deadlock_来强制在加锁前进行死锁检测



## 6. 总结

PessimisticTransaction就是一个事务的标准实现，之后再细扣下实现细节，以后提到事务的时候终于不再只是理论上的谈谈了，有了实践体会会更深。

日常PS：DB全局递增Sequence之前觉得不起眼，谁都能想到的东西，现在来看感觉真是一个NB的设计，大大简化了Snapshot和Transaction的实现。