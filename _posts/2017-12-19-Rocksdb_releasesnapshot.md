---
layout: post
title: 【Rocksdb实现分析及优化】recompact bottommost files after releasesnapshot
---

刚看完rocksdb v5.8代码，今天随手pull了一下，发现8小时前又更新到v5.9.2了，也太快了。。。这代码追的我好累好心塞~~~~(>_<)~~~~，不过v5.9.2还是有很多的Public API Change和New Features，其中有一个感觉还挺有用：

>Upon snapshot release, recompact bottommost files containing deleted/overwritten keys that previously could not be dropped due to the snapshot. This alleviates space-amp caused by long-held snapshots.

因为bottommost level中的sst相对比较稳定，如果在当时生成这些sst时存在某些snapshot，导致很多kTypeDelete类型的key全都保留并写入到文件中，那么这些key今后很难再会被drop掉（即使当初占着它的snapshot后来被release），额外占用磁盘空间。

为了解决这个问题，v5.9.2会在ReleaseSnapshot后主动检查一下bottommost level中哪些sst文件的largest_seq小于当前的oldest_snapshot，有的话将他们标记成需要compact。

原理就是这个，下面简单说下实现

## 1. 实现

```cpp
void DBImpl::ReleaseSnapshot(const Snapshot* s) {
  const SnapshotImpl* casted_s = reinterpret_cast<const SnapshotImpl*>(s);
  {
    InstrumentedMutexLock l(&mutex_);
    snapshots_.Delete(casted_s);
    uint64_t oldest_snapshot;
    if (snapshots_.empty()) {
      oldest_snapshot = concurrent_prepare_ && seq_per_batch_
                            ? versions_->LastToBeWrittenSequence()
                            : versions_->LastSequence();
    } else {
      oldest_snapshot = snapshots_.oldest()->number_;
    }
    // --- 分界线 ---
    for (auto* cfd : *versions_->GetColumnFamilySet()) {
      cfd->current()->storage_info()->UpdateOldestSnapshot(oldest_snapshot);
      if (!cfd->current()
               ->storage_info()
               ->BottommostFilesMarkedForCompaction()
               .empty()) {
        SchedulePendingCompaction(cfd);
        MaybeScheduleFlushOrCompaction();
      }
    }
  }
  delete casted_s;
}
```

分界线上面不多说，和之前差不多，就是释放对应的snapshot并且找到当前最老的快照oldest_snapshot。然后将它传入UpdateOldestSnapshot，尝试在bottommost level找到可以做compact的文件，如果有的话，就Schedule。

重点看一下UpdateOldestSnapshot：

```cpp
void VersionStorageInfo::UpdateOldestSnapshot(SequenceNumber seqnum) {
  assert(seqnum >= oldest_snapshot_seqnum_);
  // 更新oldest_snapshot_seqnum_
  oldest_snapshot_seqnum_ = seqnum;
  // 如果oldest_snapshot_seqnum_比之前记录的因为snapshot占用导致
  // 所有不能drop干净的sst文件中最小的largest_seqno大，证明当前有sst在
  // ReleaseSnapshot后被露出来，找到并将它们标记成带compact
  if (oldest_snapshot_seqnum_ > bottommost_files_mark_threshold_) {
    ComputeBottommostFilesMarkedForCompaction();
  }
}

void VersionStorageInfo::ComputeBottommostFilesMarkedForCompaction() {
  bottommost_files_marked_for_compaction_.clear();
  bottommost_files_mark_threshold_ = kMaxSequenceNumber;
  for (auto& level_and_file : bottommost_files_) {
       // 当前sst没有在做compact
    if (!level_and_file.second->being_compacted &&
       // 当前sst的最大seqno不为0 <<<--- 一会解释这里
        level_and_file.second->largest_seqno != 0 &&
       // 当前sst中至少有一个delete record
        level_and_file.second->num_deletions > 1) {
      // largest_seqno might be nonzero due to containing the final key in an
      // earlier compaction, whose seqnum we didn't zero out. Multiple deletions
      // ensures the file really contains deleted or overwritten keys.
      // 如果largest_seqno小于oldest_snapshot_seqnum_，标记成待compact
      if (level_and_file.second->largest_seqno < oldest_snapshot_seqnum_) {
        bottommost_files_marked_for_compaction_.push_back(level_and_file);
      } else {
      // 否则更新bottommost_files_mark_threshold_
        bottommost_files_mark_threshold_ =
            std::min(bottommost_files_mark_threshold_,
                     level_and_file.second->largest_seqno);
      }
    }
  }
}
```

逻辑也不难。补充一点，貌似这个只解决了DeleteRecord带来的额外空间占用，对于同一个key的频繁覆盖写不会主动发现并标记compact，如果要支持的话，可以在InternalKeyTablePropertiesNames中加一个item来记录覆盖写的record个数，然后在上面的if中判断一下。

接下来重点看下这行

```cpp
level_and_file.second->largest_seqno != 0
```

sst文件的largest_seqno怎么可能等于0？？？即使有为什么这里可以忽略呢？

原来在compact 生成bottommost level sst时，有了新的逻辑，具体如下：

```cpp
void CompactionIterator::Next() {
  ......
  PrepareOutput();
}
```

在CompactIterator执行Next的最后，多了一个PrepareOutput，它是干嘛的？

```cpp
void CompactionIterator::PrepareOutput() {
  // Zeroing out the sequence number leads to better compression.
  // If this is the bottommost level (no files in lower levels)
  // and the earliest snapshot is larger than this seqno
  // and the userkey differs from the last userkey in compaction
  // then we can squash the seqno to zero.

  // This is safe for TransactionDB write-conflict checking since transactions
  // only care about sequence number larger than any active snapshots.
  if ((compaction_ != nullptr &&
      !compaction_->allow_ingest_behind()) &&
      ikeyNotNeededForIncrementalSnapshot() &&
      bottommost_level_ && valid_ && ikey_.sequence <= earliest_snapshot_ &&
      (snapshot_checker_ == nullptr || LIKELY(snapshot_checker_->IsInSnapshot(
        ikey_.sequence, earliest_snapshot_))) &&
      ikey_.type != kTypeMerge &&
      !cmp_->Equal(compaction_->GetLargestUserKey(), ikey_.user_key)) {
    assert(ikey_.type != kTypeDeletion && ikey_.type != kTypeSingleDeletion);
    ikey_.sequence = 0;
    current_key_.UpdateInternalKey(0, ikey_.type);
  }
}
```

如果CompactIterator当前的key->value值满足以下条件：

1. 在做bottom_level_  compaction
2. 合法(valid)
3. 对应seq小于当前最小snapshot的seq
4. 不是MergeType
5. 不是当前compact inputs中最大的user_key （？WHY？？）

就将这个key的sequence置为0。

这里主要就是说如果某条record当前已经足够稳定，后期基本不会被drop，那么就将它seq置为0，对应的，如果某个sst的largest_seq = 0，就说明这个sst也足够稳定了，后期基本不需要再进行compact了。

另外，ikeyNotNeededForIncrementalSnapshot()这个感觉很鸡肋，对应的是这个：

>`DBOptions::preserve_deletes` is a new option that allows one to specify that DB should not drop tombstones for regular deletes if they have sequence number larger than what was set by the new API call `DB::SetPreserveDeletesSequenceNumber(SequenceNumber seqnum)`. Disabled by default.

没太想到这玩意的使用场景是什么。。。



## 总结

这里的确是一个优化点，我也得想想针对有TTL的sst，怎么样可以用类似的方法来主动发现并回收。