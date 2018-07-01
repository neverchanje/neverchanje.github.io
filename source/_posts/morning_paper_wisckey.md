---
title: Morning Paper - WiscKey, Separating Keys from Values in SSD-Conscious Storage
date: 2018-6-24
type: "post"
comments: false
tags: morningpaper
---

记录一些 yy 的想法，不值一提。

## vlog

对分布式 KV 来讲，能不能把 vlog 用作为 paxoslog？

我所知的两种基于 rocksdb 的典型 KV store 是这样做的：

- 像 tikv 这样的每次写数据要先写 raftlog，然后写 rocksdb WAL，最后写 rocksdb memtable。一条 value 相当于写 3 次。由于所有 replica 共享一个 rocksdb 实例，所以必须要维护 raftlog。

- pegasus 和 tikv 类似但是关掉了 rocksdb WAL，只写 paxos log。当 rocksdb memtable flush 到 L0 的时候，相应的 paxos log 才可以被 GC。这种方案需要改 rocksdb 代码，但会优于 tikv，一条 value 只需要写 2 次。pegasus 的每个 replica 都独有一个 rocksdb 实例。

现在 wisckey 来了，先写 vlog，再写 LSM index，总的数据只需要写一次。如果用在 tikv 上，WAL 可以省掉，只需维护 raftlog 和 vlog，数据只需写 2 次。如果用在 pegasus 上，paxos log 和 vlog 的合并需要一些改动，因为：

1. wisckey 的 delete 不写 vlog，只改 LSM index，为了用作 paxoslog，delete 可以改为写 tomb record 到 vlog 中。

2. wisckey 的 vlog 不维护顺序性，在 GC 过程中，一些 record 可能会从 tail 移到 head。所以需保证未提交的日志不可被 GC。

如果能够合并，那数据总的只需要写一次。不过从工程角度讲，合并 paxos log 和 vlog 应当还有更多细节。

## 稳定性

然后我们看看使用 wisckey 是否能获得稳定性提升：

rocksdb 有个问题就是当 writes 过多的时候容易出现 **write stall**，意为后台来不及 GC 导致 L0 文件数接近上限（`level0_file_num_compaction_trigger`），只好延阻写。众所周知 L0 的 read 最坏需要遍历所有文件，所以 L0 的文件数上限也不能过大。

由于这个问题，rocksdb 在大量写的场景写需要调整业务，使用某种读写分离的方案。

Wisckey 能够解决这个问题：因为 wisckey 高吞吐读写时完全可以不做 GC，最多是空间消耗大，而 rocksdb 不做 GC 则会直接影响读性能。

当然 wisckey 的后台 GC 也会影响前台工作，这点需要工程实践的权衡。

## Inline Storage
LSM 在小 value （通常小于 128 bytes）的场景下表现优于 wisckey，然而在大 value 下表现劣于 wisckey，所以可以把短 value 直接存在 LSM tree 里。如果 DB 内全都是短 value，那么它就完全退化成 LSM。如果 value size 普遍是大 value，那么 wisckey 的性能优势就能体现出来。

这种方案完美地融合 LSM 和 wisckey 的长处，工业价值很高。
