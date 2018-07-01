---
title: consensus-yaraft
date: 2017-8-3
type: "post"
comments: false
tags: raft
---

## Introduction

consensus-yaraft 是一个基于 Raft 协议 [6] 的分布式组件，它使用 C++ 实现，目的是消除 pegasus [2] 对 zk 的依赖。

我们首先对 Raft 状态机进行解耦，开发了 yaraft [3]。它仅实现了协议部分，而将网络和存储部分作为接口给上层实现，这种方式可以方便我们抛开存储和网络，对协议的实现进行充分测试。在 yaraft 的基础之上，我们对 RPC 和 WAL 进行了实现，这就是现在的 consensus-yaraft。

当前项目尚未投入使用，但一直以开源项目[1] 的形式持续在开发中。

## yaraft

yaraft 是 Raft 协议模块的实现。yaraft 起初是从 etcd/raft [4] 移植而来。etcd 作为一个被广泛使用的开源项目，其代码质量相对较高，单元测试也相对充足。我们除了移植 etcd/raft 的代码，也移植了它的单元测试，从而保证 yaraft 同样具有相对高的鲁棒性。

一般实现 consensus 有这样一种思路：节点发消息的过程称作一个 ConsensusRound，发出消息后，异步地等待结果。比如 leader 发日志，它首先广播 AppendEntries，然后异步等待。RPC 线程收到 AppendEntries Response 后提醒 leader。如果 leader 收到过半的结果（可能是过半的拒绝，也可能是过半的同意），则该轮 ConsensusRound 结束。

这里涉及到一个 ConsensusRound 生命周期管理的问题，一个 leader 如何异步等待的问题。这些还算简单（事实上 kudu[7] 的 consensus 就是这样实现的）。

还有一个问题是如何进行测试，如何构建一个本地 raft 集群。强行起多个进程，自然非常麻烦。构建一个 network simulator，节点间通过 network simulator 传输消息也是一个办法，不过这需要一些工作量。network simulator 的工作很有意义，但它的问题在于，假如我们想要模拟节点在接收到特定顺序消息时的行为时，整个过程非常麻烦。

不理解这一点的，可以在 https://raft.github.io 的 visualization demo （这就是一个 network simulator）上尝试让某一节点写入你所想要的日志序列。尝试一番你就会知道，这并不是随心所欲的。比如要写 {index:1, term:1}, {index:2, term:2}，你可能要让一个 leader 节点先发来 {index:1, term:1}，然后重新选举，term 升高为 2，再发送 {index:2, term:2}。

当然这些只是我个人的理解，我并没有实现过这样架构的 consensus，我只是想表明，yaraft 的设计可以解决这些问题，而且更简单。

raft、paxos 协议的目的是解决异步网络下的一致性问题。在异步网络中，消息的先后顺序不定，唯有的是因果关系。这种场景下适合使用状态机来解决问题。所以在 yaraft 中，我们采用这样的架构：消息驱动状态机转移，状态机会产生新的消息发送给其他节点。

这种架构面对上面的问题就有很大的优势了：要给某个节点写入 {index:1, term:1}, {index:2, term:2}，我们只需要创建一个节点，不需要 network simulator，不需要集群，然后对它发消息：

```
{type: MsgApp, entries: [{index:1, term:1}, {index:2, term:2}
```

yaraft 的大致工作流程如下：

Raft 状态机接收到消息，经过一段状态转移后，产出一系列输出。

其中一些输出指的是需要持久化的状态，这些输出会存在 HardState 中。对每个产出 HardState 的状态转移，我们都要在转移后将 HardState 的内容持久化。在持久化之前，该转移不被认为成功。

![image alt text](http://og0xhkmh3.bkt.clouddn.com/consensus-yaraft/overview/image_0.png)

HardState 包括了 currentTerm，votedFor，还有 committedIndex

状态机还会对一些请求类的 RPC 产出响应 RPC。例如在接受 MsgApp 消息后，状态机会发回 MsgAppResp；收到 MsgVote，也会发回 MsgVoteResp。响应产出后，我们需要将它们发到指定的对端节点。

日志项会先写入 Unstable，只有这些日志项持久化了，才会写入 MemoryStorage。

![image alt text](http://og0xhkmh3.bkt.clouddn.com/consensus-yaraft/overview/image_1.png)

我们如何使用 yaraft：首先通过调用 RawNode::Step 来驱动状态机转换，通过 RawNode::GetReady 来获取即将被持久化和网络传输的数据。

在获得 Ready 对象之后，用户首先将 Ready::entries 和 Ready::hardState 分别进行持久化，然后将 Ready::message 发送到其他节点上。此次请求处理即可视作完成。

## RPC

这里的 RPC 指的是两个 Raft 节点之间的网络传输。在 consensus-yaraft 中，我们基于 baidu/sofa-pbrpc[5] 实现 RPC 层。

## 异步通信模型

考虑一个问题，leader 向 follower 发送 AppendEntries 请求，follower 收到后进行状态转移，数据持久化，然后发回 AppendEntries Response，整个过程 leader 同步阻塞。而 Raft 假设整个网络是异步化的，所以我们进行一点修改：

leader 向 follower 发送 AppendEntries，follower 接收到消息后，经过状态机处理，即发回 consensus.rpc.pb.StepResponse（简称 Reply），其中包含状态码等信息。此时 leader 不等待 follower 的数据持久化，只是异步等待 Reply 发回。（TODO：暂时没有想要 Reply 报错时如何处理）

在 leader 收到 follower 发回的 AppendEntries Response 后，leader 也应该发送 Reply。由此整个通信过程完成异步化。

## Background Timer

Raft 协议需要计时器来定期执行心跳和选举超时的判断。设计上我们需要一个后台计时器，每隔 100ms 驱动状态机执行 1 次 RawNode::Tick 的操作。

考虑到多 Raft 节点部署的场景，BackgroundTimer 设计上支持多个 raft 节点共享一个 timer。

## RaftTaskExecutor

为了避免不必要的锁竞争，我们让 Raft 的状态机永远是单线程运行。timer，rpc，wal write 都会把任务放到 RaftTaskExecutor 执行，同一时间只有一个任务正在执行。

RaftTaskExecutor 含有一个 TaskQueue，这是真正工作的任务队列，我们使用 moody camel 的并发队列实现[8]。

考虑到多节点部署在同一机器上，如果每个节点都需要一个 TaskQueue 线程，那 100 个节点占 100 个线程是十分浪费的。所以我们允许多个节点使用一个 TaskQueue。正确的使用姿势是每个 CPU 分配一个 TaskQueue，不过我们现在还没考虑这个问题。

![image alt text](http://og0xhkmh3.bkt.clouddn.com/consensus-yaraft/overview/image_2.png)

## ReadyFlusher

最开始处理 Ready 的逻辑是，每次 RaftTaskExecutor 处理任务，就判断是否有新的 Ready 生成，如果有的话，就异步进行处理（推到线程池，这是 “push” 的思路）。

ReadyFlusher 最初的设计目的很简单，我认为 Ready 应该用 “pull” 的方式，从 raft node 上拉取下来。一方面这样可以让多份消息只写一次 Ready，一方面 ReadyFlusher 可以起到调配系统资源的效果，从而最优化处理 Ready。当然现在实现上比较差劲，而且 “pull” 和 “push” 的效果不见得有太大分别。

比较意外的 bonus 是 Ready 处理这部分的逻辑从 RaftTaskExecutor 上解耦，设计和单测上都舒服很多。

因为多节点部署问题，ReadyFlusher 也支持多节点共享。

## WAL

架构上我们将基于 Raft 的分布式存储分为两个部分，一个是 DataStore（比如内存的 KV 键值对存储），一个是 LogStore。日志即 WAL。节点间的 WAL 通过 Raft 协议保持一致。

每个 AppendEntries 都会携带一串 log entries 来到指定 Raft 节点，然后该节点将其写入 WAL。为了避免 WAL 无限增加，通常需要进行日志清理。一种清理方法是将 DataStore 的内容打成快照，然后将对应 WAL 删除，这种方法称作 Snapshotting。现在我们只考虑这种方案。

![image alt text](http://og0xhkmh3.bkt.clouddn.com/consensus-yaraft/overview/image_3.png)

## 日志后缀随机读写

在 Raft 协议中，由于网络分区等原因导致的日志不一致，follower 的日志后缀可能被 leader rewrite。rewrite 的过程涉及到一次随机读，定位到 conflicting log index 所在的文件 offset，然后执行 truncate 操作截断文件后缀，再将新日志项写入。

![image alt text](http://og0xhkmh3.bkt.clouddn.com/consensus-yaraft/overview/image_4.png)

实践证明这种方案不光性能差（涉及到随机读写），而且实现也比较麻烦（看上去简单）。所以与其如此，我们考虑一种优化方案。

![image alt text](http://og0xhkmh3.bkt.clouddn.com/consensus-yaraft/overview/image_5.png)

实际上我们并不需要删掉 conflicting log entries，我们可以保留它们，然后将新的日志 append 到末尾。

可能会有人认为这样不符合 Raft 的正确性，其实不然。在我们的设计中，所有的 WAL 都会同时存在内存（被称为 MemoryStorage）和磁盘中。WAL 在内存中的存储不用考虑随机读写，怎么正确怎么来即可。状态机在任何时候，都只会从 MemoryStorage 中读取日志，所以不用担心磁盘存储对正确性的影响。磁盘存储的 WAL 只负责 recovery。这样，如何从磁盘中正确地读取 WAL 到 MemoryStorage 中，就成为了正确性的关键问题。

对于上图 follower 的 [1, 1, 1, 2, 3, 3, 3, 4] 日志集，在 recovery 时，我们可能发现 {index=7, term=3} 的日志项的 index 比 {index=5, term=4} 的日志项的 index 小，这是不符合 Raft 正确性的，所以我们认为 {index=7, term=3} 是坏的，可以删掉。依次倒推，{index=6, term=3} 和 {index=5, term=3} 的日志项也是坏的，同样删之。最终我们可以读到正确日志集 {1, 1, 1, 2, 4}。

## 日志前缀随机读写

日志肯定需要被清理，如果要删除日志文件的前缀，也会涉及到一次随机读写。

![](http://og0xhkmh3.bkt.clouddn.com/consensus-yaraft/overview/image_6.png)

首先我们肯定只会清理掉 committed 的日志项。首先把 uncommitted 的日志项复制到新文件中（WAL-02），然后将原日志文件删除（WAL-01）。这种做法首先要定位 first uncommitted log entry 的文件 offset（这涉及到一次随机读），然后一次数据复制和一次数据删除。

很显然这种方式性能差，实现也麻烦。因此我们采用 segmentation 的方式。

![](http://og0xhkmh3.bkt.clouddn.com/consensus-yaraft/overview/image_7.png)

日志集写入过程中，会被分成多个固定大小的 segment。每填满一个 segment，数据就会被塞到新的 segment 中。

在日志清理时，我们只需要删掉那些 committed 的 segment 即可。

## 集成测试

有必要说一下集成测试。
consensus-yaraft 的集成测试大致是这样：首先建立一个本地集群，任意为其写数据，或者杀死某一节点，然后检查状态是否正确。
本地集群采用多进程模型，避免线程之间互相影响。对于集成测试使用多线程是否足够？答案是否定的。首先多线程 raft servers 需要共享 gflags，这点在实现上会造成麻烦。其次是我们总有一天需要进行多进程下的测试，与其现在使用”简单的”多线程模型，不如早点为多进程模型铺路。
为什么我们会对多进程产生烦恼？其实是因为 C++ 缺少一个好用的进程库（类似于 python subprocess），而从头写一个进程库并没有我们想象中那么简单（参照 https://github.com/cloudera/kudu/blob/master/src/kudu/util/subprocess.h ）。基于这个现实，我们使用 python 实现集成测试。未来类似 failover test 也可以用 python 实现。
使用 Python 的坏处是对开源用户不太友好，需要多进行一些配置。然而作为开发者是很爽的。

## References
[1] consensus-yaraft, https://github.com/neverchanje/consensus-yaraft
[2] pegasus, git.n.xiaomi.com/pegasus/pegasus
[3] yaraft, https://github.com/neverchanje/yaraft
[4] etcd/raft，https://github.com/coreos/etcd/tree/master/raft
[5] baidu/sofa-pbrpc, https://github.com/baidu/sofa-pbrpc
[6] Ongaro, Diego, and John K. Ousterhout. “In search of an understandable consensus algorithm.” USENIX Annual Technical Conference. 2014.
[7] kudu/consensus, https://github.com/cloudera/kudu/tree/master/src/kudu/consensus
[8] cameron314/concurrentqueue, https://github.com/cameron314/concurrentqueue

