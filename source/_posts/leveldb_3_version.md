---
title: leveldb 源码解说(3)：Version 和 VersionSet
date: 2016-12-11
type: "post"
comments: false
tags: leveldb
---

在阅读这篇文章之前希望读者先阅读 [rocksdb wiki 关于 Version 和 VersionEdit 的介绍](https://github.com/facebook/rocksdb/wiki/MANIFEST)，其实很多东西在这里已经介绍的很清楚了。

---------

Version 和 VersionSet 是 leveldb 最乱的地方之一。看的时候要保持思考保持清醒（这是个软件工程问题）。先看 `Version::Get`。

## Version

Version 管理磁盘上所有文件的元信息（简单说就是所有 SSTables 的 metadata），比方说哪一层有哪些文件，并且为了提升效率，我们要在内存中维护这些文件的 FileMetaData，这样我们就不用每次都去读文件才能得知每个文件的 key range。这些元数据交给 Version 来维护。每次删掉一个文件，都会导致 Version 被改变，产生出一个新的 Version。能够表现当前 DB 状态的 Version，被称作 current version 。所有 Version 都被 VersionSet 用一个循环双向链表维护起来。

其实这就有点类似 MVCC 了。MVCC 是数据库解决并发问题的时候一个很常见的概念。利用版本号的一种实现是，并发的时候数据行有多个版本用链表串在一起，写的时候拷贝一份快照出来写，加上版本号之后就变成新版本插到链表里，每个读事务只能读比自己版本号更小的，所以本质上读的时候可以进行写操作。（参考 张帅同学在知乎的回答）

leveldb 也需要类似的思想。我们后面会讲到 leveldb compaction 的机制，简单说就是 leveldb 时不时的会对一些文件进行清理（可以看成是做压缩），然后把清理干净后的数据放到新的文件集合里，这样就导致同一份数据有两批文件，旧的那一批按道理应该删掉，但是如果现在有读事务在旧的那批文件上，则暂时还不能删。所以我们利用引用计数的机制，只要一个 Version 还存活着，那么它管理的所有文件都不会被删，而一旦 Version 生命周期结束，则它管理的所有文件引用计数全部减一（参考 Version::~Version），当没有一个 Version 引用这个文件，那这个文件就可以直接删了。在引用计数这一方面，还要考虑多个用户访问这个 Version 的情况。也就是说，除了文件要引用计数，Version 自己也要引用计数。

这里有一点要注意，memtable 不归 current version 管理。只有磁盘上的数据才归 Version 管理。不过当然，memtable 本身也有引用计数。所以在 DBImpl::Get 的时候，current，memtable，im-memtable 的引用计数都会加 1，在结束的时候减一。这样就保证读的时候 memtable 不会突然被删。

## Version::Get

`Version::Get()` 查询一个 key 对应的 value，这是 leveldb 查询 key 的核心过程（划重点）。了解这步操作之前，我们首先需要知道 leveldb 的大致执行流程：首先数据会先写到 memtable，当 memtable 超过一定大小限制，数据会被持久化到 L0，为了避免在写到 L0 的同时 memtable 不可写的情况，memtable 会专门拷贝一份到 immutable memtable，顾名思义是只读的，相当于一份快照，然后利用 immutable memtable 把这份快照写到 L0。

执行到 `Version::Get` 这步之前（参照 `DBImpl::Get`），我们已经得知 memtable 和 immutable memtable 中没有对应 key，所以我们从 SSTable 中查（这里说的不够准确，后续再提）。查询过程显然是从上到下（level 0 -> level 1 -> … ）逐层遍历，这是因为 LSM tree 已经定义越新的越处于上层，所以我们一定是选择版本最新的，即含有 key 的最上层。

对于每一层的查询，首要的一个逻辑是定位 key 在哪一个文件，然后定位 key 在这个文件的位置，最终获得 key 的 value。

1. 在 level 0 定位 key：所以我们可以按照 file number 从大到小排序（越大的越新）然后依次遍历，每个文件都会有一个 key range（参照 FileMetaData），如果我们的 key 在该 range 里，则我们尝试在该文件里二分查找这个 key，如果 key 不在 range 里则 continue。通过 key range 过滤只是一种比较简单的优化，在 L0 的查询终究是比较慢的，所以 L0 的文件数必须要足够少（最坏情况下每个文件都要读一遍）。
2. 在 level n (n>0) 定位 key：在 level n > 0 上，SSTables 之间的 key ranges 不会相交，我们可以利用这点进行二分找到指定文件 （参照 version_set.c:FindFile），然后在文件里进行二分找到 key（参照 TableCache::Get）。
3. 如果找到的 key 对应的是 Delete record，那说明这个 key 已经被删了，返回 Status::NotFound。

`Version::Get` 最后会返回 key 对应的 value，key 所在的文件（GetStats）。

## VersionSet::LogAndApply

顾名思义这个函数是用来把 VersionEdit 加到 Version 里成为新的 Version。同时这个过程也会持久化到 Manifest 上。由于涉及到磁盘 IO，不可能在每次 version 改变的时候都执行一次 LogAndApply，通常是批量做。因此，VersionEdit 中包含了一系列对 Version 的修改，包括添加了某个文件，删除了某个文件。把 VersionEdit 加到 Version 里的过程本质和 git commit 是一样的，都是旧版本经过一系列修改之后产生新版本，使用 VersionSet::Builder 执行这段过程，算法很简单，实现看起来却很复杂。

写 MANIFEST 的过程我们知道就是一个写日志的过程，每条 record 意味着有一组的元数据被改变， 每组改变用 VersionEdit 构造，然后 `VersionEdit::EncodeTo` 写到 record 字符串，最后用 `log::Writer::AddRecord` 写到日志里。VersionEdit 的内部构造会保证 record 的 compactness（record 里的元数据是 key value 表示，key 是元数据类型，使用 enum 标记，用 varint 存储，不同类型的元数据 value 存储方式不一样，使用 varint 和 varstring 结合存储）。

这里可以总结一下，有下面几个操作会执行 LogAndApply：

1. memtable 刷到 L0 的时候（`DBImpl::CompactMemtable`），VersionEdit 会在这里记录新的 wal 编号，将 L0 的 FileMetaData 维护起来。
2. 进行 compaction 的时候。VersionEdit 会记录 compaction 导致的文件增加和删除操作。

