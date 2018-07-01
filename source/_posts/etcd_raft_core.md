---
title: Etcd Raft Libary 源码阅读：Core
date: 2017-1-30
type: "post"
comments: false
tags: raft
---

## Design

Raft 的基本算法（我们简称 Basic Raft）其实大家都很清楚，已有的博文不要太多，不过老生常谈的知识点大部分集中在 log replication 和 leader election。所以我在这里点一下 consensus: bridging theory and practice（后面我们称 raft thesis） 对 In Search of an Understandable Consensus Algorithm（后面我们称 raft paper）的扩展，还有一些比较少谈到但是对工程实践非常重要的东西。

## Leadership transfer (raft thesis 3.10)

具体场景就不说了，简单说，我们要把现在的 leader 换下来，换一个新的 leader 上去。其实这是一个蛮开放的问题，我们可以先放一个 follower 加入集群，然后把 leader 关掉。这样在 electionTimeout 之后集群会进行选举。按照 raft thesis 的说法，这会造成 electionTimeout 期间服务不可用。假如我们有频繁的 leadership transfer 的需求，那这确实是一个问题。

我想到一个方案是，先写一个日志下去把 electionTimeout 改小一点（比如 10s 改为 1s），然后在 leadership transfer 成功之后，把 electionTimeout 改回来。这种方案应该是不可行的，electionTimeout 不应该支持在线更改。

另一方面，我们可能需要指定（钦点）一个 follower 作为新 leader，而走 leader election 流程选出来的 leader 本质上是随机的。

raft thesis 提供的方案是，首先暂停接受 client request，然后让一个（应该是指定的） follower 的日志与 leader 完全同步，随后立刻让 follower 发起 election，由于 term 更大，旧 leader 自然被换下来。这种方案的漏洞是要保证 leader transfer 时，除了我们指定的 follower，没有其他 follower 可以发起选举。这点可以用 prevote 避免，所以 leader transfer 理论上是可以全自动化的（不需要 sysadmin 参与）。并且这种方案的前提是 follower 与 leader 完全同步的平均时间远小于 electionTimeout，当然多数情况是这样的。

## Finding the cluster (raft thesis 6.1)

这部分其实在 raft paper 里讲的很清楚了，client 发送请求给 cluster 中任意一台 server，如果这台 server 不是 leader，它就返回 leader 的地址。但是我们如何知道 cluster 里有哪些 servers，它们的 ip 是多少？raft thesis 这里提到，可以用 DNS 的方案。

## Routing requests to the leader (raft thesis 6.2)

我们在 Etcd 源码阅读：Discovery 提到过关于 zk leader 出现网络分区时，新 leader 选举出来，client 仍然连接旧 leader 的问题。raft 这里给出了解决方案。

## Linearizable semantics (raft thesis 6.3)

Basic Raft 没有 session 的概念。为什么 Raft 需要 session？

![client-session](http://og0xhkmh3.bkt.clouddn.com/raft/raft_session.PNG)

Raft 协议下的 client-server 模型是有状态的，因此每一个写请求都不是幂等的，写操作1 和 写操作2 可能数据一样，但是它们会被 raft 当成是两个不同的写操作，它们的 index 不同。所以我们需要引入 session，我们只有在 session 内才能做去重。

Further Reading：[Linearizability versus Serializability](http://www.bailis.org/blog/linearizability-versus-serializability/)

## Read-only query (raft paper 8, raft thesis 6.4)

我们知道一个写请求经过 raft 会走一遍 log replication 流程： 写磁盘，分发，过半提交后写成功。但是读请求按道理不需要走日志（就好像 MySQL 读请求不会写 binlog），所以我们需要有专门一个流程。

client 发起读操作的时候，希望能够读到最新记录。所谓最新的意思是，在读请求发起时最后一个 committed 的记录。leader 收到读请求，发起一次 heartbeat，这里证明自己确实是 leader，根据 leader completeness 属性，leader 拥有最新记录，我们这就能回复 client 了。

这种方案可以省掉一次磁盘写开销，但是日志分发还是不可避免（heartbeat）。所以我们可以批处理连续的读请求。

## Prevote (raft thesis 9.6)

在 Basic Raft 里，follower 在 electionTimeout 没收到心跳之后会发起投票，转为 candidate。然而 follower 超过 electionTimeout 没有收到心跳，很可能是由于自己的网络问题。这时候即使它发起投票，别人也不会给它响应，等它网络恢复，成功收到 leader 的心跳，它却还是会发起投票，强制集群进行一次 leader election，即使此时集群压根不需要选举新 leader。所以我们可以使用 prevote 算法。我们可以先询问其他节点，是否愿意参与选举，如果节点能够正常感知 leader，它就不会参与选举，如果节点同样认为 leader 挂了，则参与选举。过半节点参与，我们才可以发起 leader election。

Further Reading：[MongoDB: Four modifications for the Raft consensus algorithm](http://blog.neverchanje.com/2017/01/31/morning_paper_four_modifications_mongo_raft/)

## Implementation

etcd 对 raft 的实现算是比较完整了，我不知道是不是上面的每一个功能点都会被 etcd 用到。所以我个人也是倾向于把 raft 拉出来单独作为一个项目。

### Node

这是 Raft 的入口，表示一个 Raft 节点，每个节点接收到一个消息，都会驱动一次状态机转移。

```go
// Step advances the state machine using the given message. ctx.Err() will be returned, if any.
Step(ctx context.Context, msg pb.Message) error
```

看到这个接口返回 error 的方式可以看出来，我们可以用全异步事件模型操作状态机（context.Context），也可以用简单同步模型操作状态机。

- 全异步模型依赖 golang 的 channel 机制，Raft 状态机作为一个 background thread（后面我们简称 Raft Thread，或者 Raft Routine），接收上层传来的消息（通过 channel 传递消息）。状态机接收消息，进行转移。转移后的结果会异步返回给上层（node.go:Ready）。

- 简单模型就是执行 Step 时，等待状态机转移完，再得到结果，整个过程是串行同步而非并行异步的方式。这种模型的好处就是方便移植（tikv 的 rust raft 就是移植此模型），调试简单，而且实际性能应该也不差。

当然 etcd/raft 本身是异步的状态机。状态机本身异步和上层异步操作状态机是两回事。讲起来很绕，希望读者读到后面能够理解这句话。

### Raft

etcd/raft 有一点厉害的就是它把 Raft 状态机完整地作为库来实现，其实我们都明白，这需要很长时间打磨，才能把状态机解耦出来。所以我才推崇，其他语言的 Raft 实现应该把 etcd/raft 的设计翻译过去，而不是自己从头踩坑（开源协议上也是支持这么做的）。

- 发送消息：比如 follower 节点的状态机接收到 AppendEntries，它要发送 response。这时候 follower 会把消息放入 mailbox，一个内存的消息队列，真正的消息发送交给上层执行。

### CheckQuorum

Basic Raft 下，leader 只有在收到更大的 term 时才会 step down，所以在发生网络分区时，即使在其他分区里新的 leader 已经被选举出来，而旧 leader 由于接收不到新 leader 的 heartbeat，它依然会认为自己是 leader。

CheckQuorum 机制就是，每隔 electionTimeout，如果 leader 发现少于过半的节点活跃（响应心跳），则主动 step down。

这点是 etcd/raft 自己做的优化。

### NextIndex Decrease

我们知道 Raft 里 Log Matching Property 保证日志项中间不会有 holes，即日志项连续。所以 Leader 要为每个 Follower 设置一个 nextIndex（raft paper 5.3），表示下一个要发送的日志 index，如果 nextIndex 太大（此时 follower 会返回 rejection，具体地说，follower 会在 index 大于 committedIndex 时返回 rejection），就把 nextIndex 调小一点。

调小的策略对性能有一定的影响（不过一般情况是不会有 rejection 的，所以只会影响少数情况下的性能）。

- 简单的方法可以是 nextIndex-=1，然后 retry。慢慢降下来总能对上。raft paper 其实推荐这种做法，读者可以在 raft.tla 看到这个逻辑。
- 我们知道 etcd/raft 的状态机是异步模型，发送 AppendEntries 的一方并不会等待 AppenedEntriesResp 发送回来，而是继续处理其他的消息。所以可能当我们收到 AppendEntriesResp 的时候，nextIndex-1 已经不是 rejectedIndex 了。这时候去处理这个 rejection 是没意义的，nextIndex 不需要改变。
- 有一种情况是，虽然 follower 发出 rejection 的时候，它还没有 nextIndex-1 报文，但在 leader 收到 rejection 的时候，follower 提交了 nextIndex-1 报文。这时候我们不需要尝试其他更小的值，再发一次 nextIndex，follower 一定能接受。什么时候会发生这种情况：`rejectedIndex <= matchIndex` 时。raft leader 维护 matchIndex，表示成功 committed 到状态机的最大 index。
- 对于 `rejectedIndex > matchIndex` 的情况，我们必须要将 nextIndex 调小，所谓调小，就是让 `nextIndex < rejectedIndex`。所以我们需要让 nextIndex 的值保证：`matchIndex < nextIndex < rejectedIndex`。一种方案是 nextIndex = matchIndex + 1。
- raft paper 5.3 提到，rejection 可以附加一些信息，用来提示 leader 如何调整 nextIndex。最早我想应该附加 committedIndex，表示 follower 持久化到磁盘的最大 index，后来发现其实没必要，只需要发回 follower 接收到的最大 index 即可，不用管是否持久化。如果考虑持久化，那 nextIndex 会太小。

可以看出 etcd/raft 确实在 nextIndex decrease 策略上下了点功夫，虽然我个人更支持简单的 nextIndex-=1 的方案。Log Matching Property 保证了，nextIndex-=1 永远是对的。不过 etcd/raft 的确能够在一些场景下，省去几次 RTT 开销。

另外提醒一点，如果 leader 发送的 nextIndex 在 follower 上已经有了（`index < committedIndex`），follower 不会返回 rejection，而是会告诉 leader 它的 committedIndex，leader 就会跟进 nextIndex。

## Progress

我一开始也没想到在 Raft 里头还有一个挺复杂的子状态机。

参考 [etcd/raft progress](https://github.com/coreos/etcd/blob/master/raft/design.md) 文档。

![etcd/raft progress](http://og0xhkmh3.bkt.clouddn.com/etcd/progress.PNG)

这个设计蛮有道理，不过在 raft paper 里没有描述。正常我们只需要考虑，leader 接收写请求，然后把写操作复制给 follower，如果 follower 落后太多，就发送 snapshot。但是如果写请求非常多，再碰上网络分区时，leader 可能会在 buf 里累积很多待发送消息，一旦网络恢复，可能会有非常大流量顿时发送给 follower。所以一定要做 flow control。

## TODO

[ ] 后续参考 MongoDB 对 Raft 的修改，为 raft 加入新的机制。
[ ] 未来可以看 logcabin 和 consul 对 raft 的实现设计。

