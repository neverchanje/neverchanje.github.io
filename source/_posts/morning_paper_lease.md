---
title: Morning Paper - Leases, An Efficient Fault-Tolerant Mechanism for Distributed File Cache Consistency
date: 2017-1-27
type: "post"
comments: false
tags: morningpaper
---

最近要研究 etcd，所以需要专门过一遍相关的资料。
the morning paper（我说的是 adrian colyer 的那个）给 [这篇 paper](http://web.stanford.edu/class/cs240/readings/89-leases.pdf) 写过 [review](https://blog.acolyer.org/2014/10/31/leases-an-efficient-fault-tolerant-mechanism-for-distributed-file-cache-consistency/)。

------------

## Introduction

这是一篇 1989 年的陈年老 paper。我不知道 lease 这种策略的提出具体是在什么时候，不过看这篇论文的表述，应该是在论文发表前蛮早就提出来了。

lease 的原理很简单。一般 client 想要操作某个共享资源，就要申请一次所有权，操作完之后释放所有权。每次使用资源，都要申请一次，如果频繁使用，就要频繁申请。申请所有权的过程可能比较复杂，比如需要通过 ACL，比如要写 wal 等等，这时候的开销就让人头疼了。这就有了 lease。

> A lease is a contract that gives its holder specified rights over property for a limited period of time.

lease 就是“租”资源，国内叫租约，资源使用完之后不释放，一直到租期超期才自动释放。这样只要在租期内，即使频繁获取资源，实际也只会执行一次申请操作。

## Evaluation

这篇论文其实给 lease 做了一个理论评估，看看理论上 client 要如何设置租期，才会得到更高的 benefit。

> Given a read rate of R, write rate of W, and a file shared between S caches then the lease benefit factor B is given by B = 2R/SW. And you’ll be getting benefit from a leasing scheme so long as the amount of time a client holds a lease (without renewing) is > 1/(R(B-1)).

坦白说，我尝试去看计算的过程，不过没理解。colyer 帮我们做了一个总结。未来这个结论可以帮助我们设置 lease term。

## Applicability to Distributed System

lease 可以用来缓解分布式系统里的 false sharing。我们知道 cpu 的 false sharing 指的是两个 clients 同时操作一条 cacheline 的数据。分布式系统的 false sharing 指的是：

> Short lease terms also minimize the overhead due to false sharing – where a client wants to write to a file that is covered by a lease held by another client, when in fact that client is no longer using it.

上面这段摘自 colyer。

Fault Tolerance
其实这是整篇论文里我最在意的点，但是这段的篇幅却很少。

假设 server 只有一台机器，由于时钟不同步，client 和 server 可能在不同的时刻认为 lease 过期。不管是 server 过早地认为 lease 过期，还是 client 过晚地判定 lease 过期，都会造成 client 申请资源失败，这是一种 inconsistency。所以我们需要有一个 epsilon（简称 e，e>0），申请 lease 的时间是 ts，server 的过期时间为 ts + lease + e，client 的过期时间为 ts + lease - e。

上面是 server 只有一台的情况，如果 server 有多台做 quorum（甚至要考虑跨机房），问题就会大了。论文里没考虑到这点。

除此之外，我们还需要考虑 Redis RedLock 中关于进程 hang 住导致锁分配不一致的问题，也是和 lease 有关。

## TODO

未来我要看一下 colyer 推荐的 [Scaling memcache at Facebook](https://www.usenix.org/system/files/conference/nsdi13/nsdi13-final170_update.pdf) 这篇 paper。

