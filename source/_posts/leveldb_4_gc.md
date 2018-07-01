---
title: leveldb 源码解说(4)：Automatic Compaction / GC
date: 2016-12-11
type: "post"
comments: false
tags: leveldb
---


## DBImpl::MaybeScheduleCompaction

leveldb 默认做单线程 GC，如果已有 GC 在后台运行，则不会去再开一个 GC 线程。其实复杂的策略应该支持多 GC 调度，后面我们应该可以在 rocksdb 里看到。

简单情况下 GC 可以周期性地做，比如每隔 100ms 做一次。当然这在存储引擎里是行不通的，因为存储引擎永远无法预料何时突然迎来暴风骤雨般的写请求，这时候定时 GC 可能会帮倒忙。

我罗列一下哪些地方会触发 GC（也就是触发 MaybeScheduleCompaction）：

- DBImpl::MakeRoomForWrite：后面会讲到。
- DBImpl::Open：启动的时候做一次 GC 准没错。不过我暂时没有比较有说服力的解释。
- DBImpl::Get：seek compaction 触发，后面会讲到。
- Manual Compaction。

## DBImpl::BackgroundCompaction

GC 操作入口在这里。

如果有 immutable memtable（我们简称 im-memtable） 还没刷到 L0 层，则 Ln 的 GC 推迟，先刷 im-memtable。换言之，im-memtable GC 的执行优先级比 Ln>=0 GC 要高。这是因为如果 memtable 满了（大于 option.write_buffer_size），而 im-memtable 还没做 GC，整个 DB 都会被阻塞（DBImpl::MakeRoomForWrite，阻塞用的是一个 condition variable，bg_cv_）。本质上任何 Ln>=0 层的 GC 都只是优化罢了，而 memtable 塞满造成 DB 阻塞是不可容忍的。

接下来选择哪个 level 来做 compaction 呢？后面我们在 VersionSet::PickCompaction 会讲到。

接着 GC 操作落到 DoCompactionWork。

## DBImpl::DoCompactionWork

在这里我们就会真正的做 Ln|Ln+1 compaction 了。用 MergingIterator 遍历所有的 keys，忽略那些 overlap 的 keys。最终新的数据会被写到。

值得一提的是，每次遍历一个 key，都会检查一次 im-memtable（CompactMemtable），避免 GC 花的时间太长，阻塞了 memtable。

随着程序运行，level n>=0 会做 compaction，把数据清到 n+1 层。这里要考虑 在什么时候 leveldb 会让这个 level 做 compaction？
参考 `version_set.cc:Version::Builder::Apply` 底下的一段注释：

## VersionSet::PickCompaction

compaction 的选择策略有两种，一种是 seek compaction，一种是 size compaction。

### seek compaction

这是由于某个文件的内容被过多访问（负载过大），多次 seek 的开销下，一次 compaction 的开销是可以容忍的。

```cpp
// We arrange to automatically compact this file after
// a certain number of seeks.  Let's assume:
//   (1) One seek costs 10ms
//   (2) Writing or reading 1MB costs 10ms (100MB/s)
//   (3) A compaction of 1MB does 25MB of IO:
//         1MB read from this level
//         10-12MB read from next level (boundaries may be misaligned)
//         10-12MB written to next level
// This implies that 25 seeks cost the same as the compaction
// of 1MB of data.  I.e., one seek costs approximately the
// same as the compaction of 40KB of data.  We are a little
// conservative and allow approximately one seek for every 16KB
// of data before triggering a compaction.
f->allowed_seeks = (f->file_size / 16384);
if (f->allowed_seeks < 100) f->allowed_seeks = 100;
levels_[level].deleted_files.erase(f->number);
levels_[level].added_files->insert(f);
```

我们知道 `DBImpl::Get` 可能会从 memtable 中查，查不到就在 SSTable 中查。如果在某一个文件上的查询次数太多（多于 allowed_seeks，参考 `Version::UpdateStats`），则会针对这个文件进行一次 GC。上面的注释里说到了这么做的原因：一次查询的开销相当于对 40kb 做一次 compaction 的开销，所以对大小为 x 的文件做一次 compaction 的开销，等于 x/40kb 次查询的开销。而真实情况需要更保守，leveldb 选择在 x/16kb 次查询之后，就需要做一次 compaction。

### size compaction

这是由于某层的大小过大。我们在 `VersionSet::Finalize` 对其细细描述。

当然如果两个 level 同时超过限制，两个 GC 会依次做，同一时间不会有两个 GC 在进行。GC 是一个后台线程，利用了 PosixEnv::Schedule，这是一个生产者消费者队列，当然消费者只有一个。

## DBImpl::MakeRoomForWrite

但这样在高并发写的场景下，Ln GC 可能会经常没有办法做，而 L0 会越积越多。为了防止这种情况发生，leveldb 给 L0 的大小设限（参考 config::kL0_SlowdownWritesTrigger）。当 L0 的大小超过了这个限制，在写操作执行之前，leveldb 会强制进行 Ln GC。这里 leveldb 不会让 GC 一次性全部完成，这样会导致 write latency 太大。所以我们总结 leveldb 的策略是，L0 过大的情况下，在写数据之前，先做短时间的（1ms，暂不可配置） GC。

如果当前暂时不需要做 im-memtable GC，而我们的 L0 确实过大了（大于 config::kL0_StopWritesTrigger），则在写操作执行之前，整个 DB 会暂停下来先做 Ln GC。

每次 memtable 一满，memtable 就把所有数据交给 im-memtable，然后自己创建一个新的，空的 memtable。DBImpl::CompactMemTable 用来把 im-memtable 写到 L0。当然，每次生成 L0 文件，都要用 VersionEdit 记录这次改动。

Ln 把 compaction 的结果写到 Ln+1 层，那原来的 Ln 层就需要删了。另一方面由于引用计数机制，我们不能删计数器大于 1 的文件。这就是 DBImpl::DeleteObsoleteFiles()。

Ln 和 Ln+1 两层合并的时候需要让能够 merge 的数据尽可能多，尽可能不会浪费（参考 VersionSet::SetupOtherInputs）。

## VersionSet::Finalize

size compaction 通过给各个 level 打分，评选出分数最高的 level 来做 compaction。

## Compaction::ShouldStopBefore

这里是让 Ln GC 提前结束的策略。这个策略很奇怪，避免 compaction 结果和 Ln+2 有过多重叠，

## DBImpl::WriteLevel0Table

这里其实就是把 im-memtable 刷到 L0 的过程，一开始我以为很简单，一个写文件就好了。其实不然，没有人说 im-memtable 只能被刷到 L0。我们只要保证：如果一个 key 在 Ln 和 Lm 存在，且 m>n，那么 Ln 上的 sequence number 大于 Lm 上的 sequence number。为什么不直接把 im-memtable 刷到 L0？很简单，为了减小 L0 compaction 的压力，把 GC 分散到下层，分散各层 GC 的负载。

现在问题来了，im-memtable 应该被刷到哪个 level？我们找与 im-memtable 有 overlap 的那个 level n，则插入到 n-1 层。这样就能保证上述的要求。

