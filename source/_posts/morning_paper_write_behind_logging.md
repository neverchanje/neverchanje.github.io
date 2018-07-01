---
title: Morning Paper - Write-Behind-Logging
date: 2018-6-19
type: "post"
comments: false
tags: morningpaper
---

理论上我们可以直接把数据写入 nvm，但是由于 nvm 和 dram 之间还有较大性能差距，我们会先把数据缓存在 dram，再写入 nvm。这样的架构称作 two-tier-hiarachy。

数据首先缓存在 dram 中的 DTT（dirty-tuple-table），在提交的时候，统一刷入 nvm。具体过程分为：

- write nvm table heap

- nvm sync

如果在 nvm sync 之前机器挂了，nvm 里面就会有脏数据。WBL 的做法是写一条日志表示 “我是现在最新的提交，往后 100 条写都是脏的”。（100 这个值可配）。

基于这个设计，WBL 的提交过程是：

- write nvm table heap

- nvm sync

- write WBL

- nvm sync

所以每次读的时候，需要注意不去读脏数据。如何避免脏数据？WBL 假设 DB 使用 timestamp 来标记写（MVCC）。假如当前提交至 300，那么 timestamp 在 [301, 400] 之间的写则全都会认为是脏的。

recovery 时脏的写会被 abort。对于这些脏数据，DB 在 timestamp GC 的时候可以顺便清理。

有一个严重问题是如何实现 log replication。以前的 replication 从没考虑过 NVM 这种超强存储，这样 Paxos（synchronous replication）这种先提交日志，再写 DB 的模型无法发挥 WBL 的最大威力。因为即便 NVM 效率再高，日志 WAL 也要先提交到 3 副本。相信后面会有基于 RDMA 和 NVM 的 Paxos 工作出现。

对于空间使用率上：

- WBL 不需要 checkpoint（WAL 模型下为了减少日志，加快 recovery）

- 永远只需要维护极少量 log

- DB data structure 也可以优化空间使用
