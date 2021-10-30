# Test
# Memtable

在写操作前，需要对memtable的剩余空间大小进行计算，估算其是否能容纳一次写操作的量；如果不能容纳，就需要将memtable转储为sstable，删除掉memtable并新建一张空白的memtable。

转储memtable前会进行一次**移形换位**操作，将memtable的指针赋值给imm_，重新生成一张空白的memtable，并将转储immtable成sstable的操作放在后台进行，这样可以使Memtable Compact的操作与memtable读写操作的并行；当memtable又需要compaction，而旧的immtable的compaction操作尚未完成时，等待。

```cpp
// REQUIRES: mutex_ is held
// REQUIRES: this thread is currently at the front of the writer queue
Status DBImpl::MakeRoomForWrite(bool force) {
  mutex_.AssertHeld();
  assert(!writers_.empty());
  bool allow_delay = !force;
  Status s;
  while (true) {
    if (!bg_error_.ok()) {
      // Yield previous error
      s = bg_error_;
      break;
    } else if (
        allow_delay &&
        versions_->NumLevelFiles(0) >= config::kL0_SlowdownWritesTrigger) {
      // 当level0的sstables即将到达compaction限制时，主动休眠1ms，以更多的出让CPU给
      // 后台的compaction进程。休眠完后再进行一次判断。
      // We are getting close to hitting a hard limit on the number of
      // L0 files.  Rather than delaying a single write by several
      // seconds when we hit the hard limit, start delaying each
      // individual write by 1ms to reduce latency variance.  Also,
      // this delay hands over some CPU to the compaction thread in
      // case it is sharing the same core as the writer.
      mutex_.Unlock();
      env_->SleepForMicroseconds(1000);
      allow_delay = false;  // Do not delay a single write more than once
      mutex_.Lock();
    } else if (!force &&
               (mem_->ApproximateMemoryUsage() <= options_.write_buffer_size)) {
      // There is room in current memtable
      break;
    } else if (imm_ != NULL) {
      // We have filled up the current memtable, but the previous
      // one is still being compacted, so we wait.
      Log(options_.info_log, "Current memtable full; waiting...\n");
      bg_cv_.Wait();
    } else if (versions_->NumLevelFiles(0) >= config::kL0_StopWritesTrigger) {
      // There are too many level-0 files.
      Log(options_.info_log, "Too many L0 files; waiting...\n");
      bg_cv_.Wait();
    } else {
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
      mem_ = new MemTable(internal_comparator_);
      mem_->Ref();
      force = false;   // Do not force another compaction if have room
      MaybeScheduleCompaction();
    }
  }
  return s;
}
```

memtable转储为sstable的核心逻辑在`WriteLevel0Table`中：

```cpp
Status DBImpl::WriteLevel0Table(MemTable* mem, VersionEdit* edit,
                                Version* base) {
  mutex_.AssertHeld();
  const uint64_t start_micros = env_->NowMicros();
  FileMetaData meta;
  meta.number = versions_->NewFileNumber();
  pending_outputs_.insert(meta.number);
  Iterator* iter = mem->NewIterator();
  Log(options_.info_log, "Level-0 table #%llu: started",
      (unsigned long long) meta.number);

  Status s;
  {
    mutex_.Unlock();
    s = BuildTable(dbname_, env_, options_, table_cache_, iter, &meta);
    mutex_.Lock();
  }

  Log(options_.info_log, "Level-0 table #%llu: %lld bytes %s",
      (unsigned long long) meta.number,
      (unsigned long long) meta.file_size,
      s.ToString().c_str());
  delete iter;
  pending_outputs_.erase(meta.number);


  // Note that if file_size is zero, the file has been deleted and
  // should not be added to the manifest.
  int level = 0;
  if (s.ok() && meta.file_size > 0) {
    const Slice min_user_key = meta.smallest.user_key();
    const Slice max_user_key = meta.largest.user_key();
    if (base != NULL) {
      level = base->PickLevelForMemTableOutput(min_user_key, max_user_key);
    }
    edit->AddFile(level, meta.number, meta.file_size,
                  meta.smallest, meta.largest);
  }

  CompactionStats stats;
  stats.micros = env_->NowMicros() - start_micros;
  stats.bytes_written = meta.file_size;
  stats_[level].Add(stats);
  return s;
}
```

`BuildTable`接收memtable的iterator（基于跳表实现），生成新的sstable并将sstable的元数据`FileMeta`返回。虽然memtable可以直接刷到Level 0上，但由于每个record平均合并k/2次后才会被刷到下一层，因此我们更贪心的将新的sstable向下层刷，这就需要检查新的sstable与level0的sstable间是否有overlap；如果有的话，那就直接刷到level 0，否则，期望刷到level 1，然后再检查与level 2间是否有overlap，循环往复，直到找到最终插入的层次。找到之后，一个新sstable的`FileMeta`就齐全了，这个新的`FileMeta`会被添加到VersionSet中并被持久化记录，这样重启/崩溃后就可以找到这个sstable了。

> 补正：我暂时还不知道compact操作中两个level的文件合并后会刷到哪一层，这样看来刷到哪一层都是比较有道理的，刷到下层可以减小写放大，刷到上层对于热点数据来说少一次二分查找变相减小读放大。

使用顺序写取代随机写对于机械磁盘来说是一个很好的提升，但对于SSD来说可能会有偏差。一是因为SSD的顺序写和随机写差异并不显著，因此顺序写策略的收益可能并不高；二是为了减小读放大leveldb会频繁的进行compact操作引入大量读写，而频繁读写会降低SSD的使用寿命。



下一步计划：MVCC
