---
title: leveldb 源码解说(2)：Log
date: 2016-11-16
type: "post"
comments: false
tags: leveldb
---

前面有说到 MANIFEST 用的是日志来记录，这种元数据日志肯定要做 Compaction（leveldb 的 log 文件并不是通过删除冗余日志记录来做 log 的 compaction，而是直接删除过期且无用的 log 文件，参考 `DBImpl::DeleteObsoleteFiles`），同时也要考虑具体 format（[官方文档](https://raw.githubusercontent.com/google/leveldb/master/doc/log_format.txt)）。

一般为了能快速定位数据（优化读），设计上会把日志文件切分成固定大小的 blocks，leveldb 的日志会被分成 32kb 的 blocks。照道理一个 block 可以作为一个文件， 不过这样会让文件系统管理过多的 metadata (inodes)。所以可以把多个 blocks 放在一个文件里。[?] 暂时不知道一个文件最多能放多少 blocks。

一个 `log::Writer` 负责对某一个日志执行写操作。日志的切分要考虑 boundary 问题，即考虑一条 record 可能塞不进一个 block 的剩余空间里，这样就需要把这种 record 分成 FIRST, MIDDLE, LAST 三个连续的部分（我们后面会直接称作 fragment）然后塞到多个 blocks 里（其实我觉得多数场景下应该不会有 MIDDLE，因为一条日志一般不会超过 32kb [?]），把可以被塞进一个 block 里的标记为 FULL。

官方文档给出了 block 的 format，作为磁盘文件格式已经比较 compact 了，后续优化空间不大的样子。

```
block := record* trailer?
record :=                    // 其实这里不太严谨，准确说这应该是 fragment，不是 record
     checksum: uint32        // crc32c of type and data[] ; little-endian
     length: uint16          // little-endian
     type: uint8             // One of FULL, FIRST, MIDDLE, LAST
     data: uint8[length]
```

block 的实现是比较简单的事情（说实话一般给出 format 设计之后，实现和测试都是时间问题）。这里有必要考虑 checksum 计算的问题，即 crc 计算哪部分的校验和。通常来讲是 length + type + data，这一定是对的，不过 crc 算法本身就包括了字符串长度的检验，我们可以省掉 length 这一项。所以只需要计算 type + data（crc 保证一个校验正确的长度一定是 length+1，这就保证了 length 的正确性，即 type + data 正确，length 一定正确，反之亦然）。leveldb 这里还在 crc 计算中做了小优化，即提前计算好每个 type 的 crc（对应的是 log::Writer::type_crc_），这样就不用每次都重新算一遍。

`log::Reader` 负责对某一个日志进行读操作。读日志是一次性读一个 block 的量到内存里。`log::Reader` 的实现有些复杂，因为要考虑到错误处理（主要是 Data Corruption，比如磁盘读到一半磁盘挂了，读的内容小于 blocksize， 但要与 eof 区别对待）。leveldb 设计了 `log::Reader::Reporter` 来让用户自定义错误处理，比方说统计丢失了多少字节。

```cpp
// Interface for reporting errors.
class Reporter {
 public:
  virtual ~Reporter();
  // Some corruption was detected.  "size" is the approximate number
  // of bytes dropped due to the corruption.
  virtual void Corruption(size_t bytes, const Status& status) = 0;
};
```

`log::Reader` 有个参数是 initial_offset，表示从中断处开始读日志，它会带来些比较麻烦的边界情况：

考虑到我们是一个 block 一个 block 依次读的，然而 `log::Reader` 的 `initial_offset_` 不一定在 block 的起始处，它很可能在某 block 的中间，为了方便说明，把这个 block 记作 x。我们还是按照规则把整个 x 读到内存。x 的起始处到 initial_offset 这段区间上的 records 我们都忽略掉。
读到这个 fragment 之后我们要判断它是不是 MIDDLE 或者 LAST 的，如果是的话，我们就把它忽略，这里需要把因为错误而忽略的字节数报给 reporter。
还有一些边界情况：

我们定义了 `record := FULL / FIRST + MIDDLE* + LAST`，所以如果出现

- FIRST 后面跟 FULL，报错给 reporter
- FIRST 或者 MIDDLE 后面跟 FIRST，报错给 reporter
- LAST 后面跟 MIDDLE，报错
- type 不是 FULL / FIRST / MIDDLE / LAST 的任何一个，报错
- crc 校验失败，字符串长度和实际不符，报错

在 `log::Reader` 的实现里面会看到 resyncing 这个词。在 RAID 1(mirrored) 磁盘阵列里，resync 指的是数据写的时候中途宕机，导致几个磁盘数据不一致，恢复的时候在后台线程重新进行同步，保证几个副本一致性的过程（参考）。在 `log::Reader` 里，`resyncing_` 表示我们是否是从一个中断处（>0 的 initial_offset ）开始读日志的。

