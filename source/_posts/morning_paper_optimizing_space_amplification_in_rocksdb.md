---
title: Morning Paper - optimizing space amplification in rocksdb
date: 2018-6-27
type: "post"
comments: false
tags: morningpaper
---

其实是一个比较小众的需求：我们的机器 cpu 和 ssd 处理能力充裕，实在是磁盘空间不够。所以需要想方设法把空间利用率提上来，但要尽可能少的影响读写。

这个需求建立在一个前提下：fb 有一层大规模分布式 cache 层承担绝大部分请求，所以落到 rocksdb 的 query 不多。（不知道什么 workload 非得加 cache 层）

几种策略：

## Key prefix encoding

这个是 leveldb 就有的优化，相邻 key 共享 prefix，不提。

## Tiered compression

其实就是加压缩，但是不在 level 0-2 加压缩，因为这样会影响读性能。只在 level > 2 加压缩，因为层数高的数据访问频率小。我们知道这个配置项就是 compression_per_level。

## No bloom filter for last level

最后一层的数据量比前几层加起来都多，访问频率又低，所以省掉 bloom filter block 可以省掉许多存储空间。这个相关的配置项没找到。

## dynamic level compaction

这个选项其实是 level_compaction_dynamic_level_bytes ，是一个 advance option，facebook/rocksdb 这里是这个 feature 发布时的官方博客。它的思想是从 space amplification 的理论入手来节约空间。
```
space amplification = space on filesystem / space of user data
```
考虑在一个理想稳定的 workload 下，用户数据长期稳定不变，只有少量增删和更新，此时 last level 和用户数据量相近，故
```
multiplier = 10，Lx = last level
space amplification = (L0 + L1 + ... Lx) / Lx = 1 + 0.1 + 0.01 ... = 1.11
```
然而通常 last level 填不满，对于 Lx = 2 Lx-1：
```
space amplification = (Lx + 0.5Lx + 0.05Lx + ... ) / Lx = 1.55
```
所以如果 last level 填不满，就自适应地把 level 调小，就能够接近最优空间放大 1.11。

## compression dictionary

有些压缩算法支持全局 dictionary 来提高压缩率。相关配置是 `CompressionOptions::max_dict_size`。不太懂这个。
