---
title: leveldb 源码解说(1)：Preview
date: 2016-11-12
type: "post"
comments: false
tags: leveldb
---

## Introduction

这系列博文被我叫做源码解说，但我并不想太细地描述 leveldb 的代码逻辑，我更希望能够对 leveldb 的各种实现策略进行解读。leveldb 经过这么长时间的发展，其实内部的一些策略已经足以成为教科书式的范例。我提前建议，读者最好对操作系统和数据库有自己的理解再来读。因为我自己也想用 C++11 重写一套兼容 leveldb 接口（其实不太难）的存储引擎。另一方面，我也会从一个实现者的角度来解说，而不会简单概括原理。如有错误欢迎指出。

除了我的文章之外，还有一些文章我认为比较好的资料可以阅读：

- [sparkliang 的leveldb 源码分析](http://blog.csdn.net/sparkliang/article/category/1342001)
- [rocksdb wiki](https://github.com/facebook/rocksdb/wiki)

## leveldb 介绍

在这个专题的开始，我们介绍一下 leveldb 的大致原理，这样后面的各个小节可以相互隔离。

![leveldb architecture](http://og0xhkmh3.bkt.clouddn.com/leveldb_architecture.jpg)

来自 An Efficient Design and Implementation of LSM-Tree based Key-Value Store on Open-Channel SSD。
对 leveldb 的介绍有很多了，这篇论文的 2.1节 简单介绍了一下 leveldb。

### LSM

LSM 的设计是为了减少 random writes，同时保证 acceptable read performance。这里我们给几个 LSM 的结论：

- L0 的数据是从 memtable 刷下来的（实现在 `DBImpl::WriteLevel0Table`），由于 append-only 策略，L0 的 SSTable 内部不会 overlap（所以 SSTable 全称是 Sorted String Table），因为从 memtable 刷下来的时候会做一次 compaction，但是 SSTable 之间会 overlap。

- L0 往下，每一层的 SSTable 都经过了 compaction 才刷下来的（我们应该把这叫 GC）。所以除了 L0 以外，同层之间不会 overlap，而层与层之间会 overlap。

### Glossary

- overlap 的含义这里做个简单解释：一个 key 的生命周期里，它可能会被多次 update，也可能会最终被 delete，每次 update 操作都相当于 append 一个新 record，delete 操作也是通过 append 一个 tombstone record ，把前面的操作覆盖掉，这种新版本覆盖旧版本的情况就叫 overlap，这是 append-only 的思想（相信这些对读者应该是废话）。但在 leveldb 上 overlap 的含义又做了扩展：SSTable 由于按 keys 排序，所以一个 SSTable 文件有一个 key range，如果两个 SSTable 的 key range 有交集，也叫作 overlap。

- compaction 这里也做一个解释：我们把旧版本操作去掉，只留下最新的，这就是 compaction。

## 从 db_bench 开始

由于工作原因，我需要去阅读一些项目源码，知乎上有些朋友认为阅读源码先去阅读 main 函数，对大概流程有个了解之后再去针对模块阅读。这本质是对的，但是我们仍然可以有一些取巧的方法，比如读 benchmark 的代码，比如读单元测试的代码。这些经常被忽略的东西其实是阅读源码很好的切入点，能给读者所见即所得的过程。

我们从 main 函数开始看，这里用的是非常原始的命令行手动解析（连 `--help` 都没有）。其实 gflags 做 command line flags 还是蛮好用的，不考虑依赖的话应该用 gflags。

如果不指定的话（`--db`），leveldb 的日志文件数据文件都会放在 `/tmp/leveldb-%d/dbbench` 下。

```
➜  dbbench ls -lh | awk '{print($5,$9)}'
3.3M 000071.log
2.1M 000080.ldb
2.1M 000081.ldb
...
2.1M 000112.ldb
2.1M 000113.ldb
16 CURRENT
0 LOCK
17K LOG
5.8K MANIFEST-000002
```

可以看到非常工整的 2.1M ldb，空的 LOCK 文件，CURRENT 的内容是 ”MANIFEST-000002“，可以推断 CURRENT 指向最新的 MANIFEST。具体的文件名定义可以在 filename.h/cc 里找到。我们可以在 google 官方文档 里看到这些文件的解释。

### leveldb 里的各种文件

- MANIFEST 的文件名对应的是 DescriptorFileName，用来持久化一些 leveldb 的 metadata。每次 leveldb 的元数据改变，不会重写 MANIFEST（因为元数据很多，每次都因为一两个元数据的改变就重写的话效率太低），而是使用日志来记录 DB 元数据的改变（append-only）。后续再细讲 MANIFEST。

- CURRENT 指向最新的 MANIFEST

- LOCK 即用来做文件锁，后面在 DestroyDB 中会介绍

- .ldb 文件是存 SSTable 的数据文件，以前叫 .sst，后来因为一些原因所以换了后缀。

benchmark 结果（这块是 SSD 和 SATA 的混合盘）：

```
fillseq      :       2.674 micros/op;   41.4 MB/s
fillsync     :    1391.578 micros/op;    0.1 MB/s (1000 ops)
fillrandom   :       4.380 micros/op;   25.3 MB/s
overwrite    :       6.616 micros/op;   16.7 MB/s
readrandom   :       7.241 micros/op; (1000000 of 1000000 found)
readrandom   :       6.575 micros/op; (1000000 of 1000000 found)
readseq      :       0.265 micros/op;  417.4 MB/s
readreverse  :       0.434 micros/op;  254.9 MB/s
compact      :  819660.000 micros/op;
readrandom   :       4.581 micros/op; (1000000 of 1000000 found)
readseq      :       0.243 micros/op;  454.5 MB/s
readreverse  :       0.381 micros/op;  290.6 MB/s
fill100K     :    1334.272 micros/op;   71.5 MB/s (1000 ops)
crc32c       :       4.285 micros/op;  911.5 MB/s (4K per op)
snappycomp   :       5.795 micros/op;  674.1 MB/s (output: 55.1%)
snappyuncomp :       0.841 micros/op; 4643.7 MB/s
acquireload  :       0.416 micros/op; (each op is 1000 loads)
```

这里的几个测试写的 benchmark 有 fill 和 overwrite。写测试结束之后默认都需要清空数据库，类似于 DROP TABLE，对应 `db_impl.cc/DestroyDB`。[?] 调用 DestroyDB 之后再把 DB delete 掉，在下次使用时重新 Open。

DestroyDB 的参数 dbname（包括 FLAGS_db）即 `/tmp/leveldbtest-0/dbbench`。删除数据库即将该目录删除。删除该目录还需要考虑并发问题。

> **文件锁**
> DestroyDB 是通过文件锁来解决并发时候的一致性问题的，相当于每次 DestroyDB 都是一次事务。文件锁有两种概念，一种是多进程场景下的，利用 fcntl 和 flock 来实现，一种是多线程场景下的，我们没有必要使用系统调用，可以直接在用户态内存中维护一个锁表（set实现，申请锁就往 set 里插入文件名，释放锁就 erase 该文件名，如果 set 中已有该文件则操作失败），这样利用简单的 mutex 来实现即可。
> 锁表其实是一种 tradeoff。加锁写时开销大，争抢锁时开销小，优化多线程 contention。加锁过程在锁表加锁，再加文件锁。释放锁就倒个顺序。

benchmark 的执行方法 `Benchmark::RunBenchmark`：每个 benchmark 都开多个线程（--thread）执行来模拟并发。最后再将多个线程的结果 merge 起来，就是总的结果。当然并发这部分未来还是用 count_down_latch 来实现，leveldb 这种写法比较丑（用一个 SharedState，本质还是 count_down_latch）。

- `Benchmark::WriteSeq / Benchmark::WriteRandom -> Benchmark::DoWrite`。

顺序写保证有多少个写操作就有多少个 entries， 随机写则不一定，有些写操作可能是 update，leveldb 的更新和插入用的都是 `DB::Put`。

> **用 DB::Write 进行写**
> 来看写过程：通过 WriteBatch，然后再用 `DB::Write` 写到数据库中，WriteBatch 相当于一个辅助类，给使用者一个批处理的接口，对用户而言，一个 entry 就是一对 Key + Value，然而对于 leveldb 而言，一个 entry 是 InternalKey + Value。WriteBatch 就是用来将用户的 entry 转化成 leveldb 的 entry。然后组成 batch 一起写到 DB 中。比较形象的形容是，对应到 MySQL 中，我们可以认为是一组 INSERT 语句，然后用一个 Transaction 写到 DB，WriteBatch 就是用来生成这一组 INSERT。

- `Benchmark::ReadSequential` 和 `Benchmark::ReadReverse` 用的是 `DB::NewIterator` 创建迭代器来遍历。

> **leveldb 的 Iterator**
> leveldb 的 Iterator 设计优势在于头文件确实干净，缺点在于不那么 modern-c++-style，用完还要 delete。未来我们可以用 `silly::IteratorFacade`（更干净的 `boost::iterator_facade`） 来做。

- `Benchmark::ReadRandom` 就是随机 Key，用 `DB::Get` 查。`Benchmark::ReadHot` 比较有意思，随机 Key 的范围更小，应该可以用来测试 cache 性能。

- 这里专门有一个 `Benchmark::SeekRandom` 的测试，用的是 `Iterator::Seek`。现在暂时还不清楚 `DB::Get` 和 `Iterator::Seek` 有什么区别。

- `Benchmark::Compact` 会对整个 DB 进行一次 Compact（相信读者已经懂 leveldb 大致原理）。这是 leveldb 的一块很硬的蛋糕，调用 `DBImpl::CompactRange`。

- `Benchmark::SnappyCompress` 只是单纯测试 snappy 算法压缩 1G 数据的效率，`Benchmark::SnappyUncompress` 也只是测试解压效率。一开始还以为是测试加了 snappy 的读写效率。

其实还有其他几个 benchmark，我们可以在 FLAGS_benchmark 加，但是我想等我们深入了解了 leveldb 再回来实际测一番。


