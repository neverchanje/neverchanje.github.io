---
title: 快速理解一致性协议 raft
date: 2017-10-25
type: "post"
comments: false
tags: raft
---

为了更好地拉人入坑，我决定写篇关于 raft 的教程出来，尽可能深入浅出一些，让甚至是没接触分布式系统的同学也能理解。paper 还是要看的，只不过这篇文章希望做一个概览。

继续广告，我的项目在：

[neverchanje/yaraft](https://github.com/neverchanje/yaraft)

[neverchanje/consensus-yaraft](https://github.com/neverchanje/consensus-yaraft)

---------------------

Raft 解决的问题很简单，就是让多个副本的日志数据达成一致。举个例子，就是地球上发生了几个事件，称作 {1,2,3}。A 是某国家主席（称为 leader），它得知事件是 {1,2,3}；B, C 是香港记者（称为 follower），它们只知道事件 {1,2}。如何让 A, B, C 对事件集合达成一致，就是 Raft 所需要解决的问题。

很简单的，leader 需要告知其他 followers，发生了事件 3，并且它发生在事件 1,2 之后。

接下来正经点说。

分布式系统里，机器时不时就会崩，网络时不时就会抖。当然如果你把 timeout 设置到 10s，出错的概率其实不会很高，但这就意味着你的系统有最高 10s 的不可用时间。

所以我们引出了几个需求：

- 数据不能丢，需要多节点备份

- 有一定的容错能力（fault tolerance），少数节点的网络抖动不会对多数节点造成影响。

这个问题发展出了一套理论，我只简单地讲讲：

1. 考虑 2N+1 个节点。一份数据，至少需要复制到 N+1 个节点上才算是提交（commit），否则可能出现冲突。比方说事件 x=1 发到 N 个节点，同时事件 x=2 发到另外 N 个节点。如果 x=1 和 x=2 同时提交了，最终 x 的值是多少很难说清。这和多线程锁是一样的。2N 个节点的情况也同理。

2. 一份数据复制到 N 个节点上，如果要容忍 K 个错误，需要 `K < N`，这样才能保证复制到 N 个节点满足数据提交的要求。也就是说，N 个节点，最多容忍 N-1 个错误。

我也时常在想一个问题，为什么多数一致性协议都要求过半节点复制成功才算提交？3 个节点，复制到 2 个才算提交（2/3）；5 个节点，复制到 3 个才算提交（3/5）。为什么没有一个专门优化 4/6 或者 4/5 的协议？

我觉得可能是“过半”这个 constraint 足够松，它能够最大程度容忍错误，能够最小程度上保证提交。

思考日志分发（log replication），可以围绕这个简单点考虑：

**leader 将日志发给 follower，过半就算提交。**

假设 5 个节点，假设 leader 永远在任，那么除非有 3 个 followers 全部失效，否则系统没有问题。follower 一切坚持以 leader 为主导，深入贯彻其基本思想，只要 leader 是对的，即便有 follower 赶不上，leader 也能把它往前拉。

然而 leader 也是会意外下马的（比如陈水扁），一旦 leader 出事，问题就来了。上一任 leader 说过的话，算不算数呢？应该听当前这一任 leader 的，还是听上一任的？

我们为每一届选举，标记一个任期 term。

```
A (Leader): <index: 1, term: 2> <index: 2, term: 2> <index: 3, term: 3>
B (Follower): <index: 1, term: 2> <index: 2, term: 2> <index: 3, term: 2>
C (Follower): <index: 1, term: 2> <index: 2, term: 2>
```

A 当选了新的 leader，他是在第三届选举当选的，所以当前 term 为 3。B，C 为 follower。A 和 B 存在冲突。A 认为地球上发生了三件事，第三件事是在 term=3 的时候发生的，而 B 认为第三件事是在上一任 leader 在的时候（term=2）发生的。

因为 A 有最大话语权，所以 B 只能乖乖听话，修改记录：

```
B (Follower): <index: 1, term: 2> <index: 2, term: 2> <index: 3, term: 3>
```

Raft 的选举也是如日志分发类似，过半节点认同候选人（candidate），它才能当选 leader。

过程大概是 candidate 发起投票，follower 可以投同意票，也可以投反对票。过半的 follower 同意，则 candidate 当选为 leader。

显然，这样每一届选举，只能选出一个 leader。

**群众要为选举出来的 leader 负责。**

假设 A, B, C 把 C 选举出来作为 leader，那么惨剧就目不忍视了：

```
A (Follower): <index: 1, term: 2> <index: 2, term: 2> <index: 3, term: 3>
B (Follower): <index: 1, term: 2> <index: 2, term: 2> <index: 3, term: 3>
C (Leader): <index: 1, term: 2> <index: 2, term: 2>
```

虽然 `<index: 3, term: 3>` 这条日志已经提交（复制过半），C 依然会无情地将它们抹去。

这是不合理的。好比如 A 和 B 在 `<index: 3, term: 3>` 确立了社会主义，结果 C 一上来就要改革为农奴制，而没有取得 A, B 的共识。

所以在 Raft 中，C 是永远不能当选为 leader 的。A, B 会认为 C 不够新。过半的节点认为它不够新，C 就不可能当选。

考虑网络的异步化，真实网络很难让我们再用现实世界举例。现实中上一任主席通常不会对当下的国家再下命令（垂帘听政的意思），而网络中却时有发生。

比如第三届选举当选的主席，却收到第二届选举发起的投票，这是不合常理的。这时候它不会去投票，而是会直接无视。

所以，term 就成了是否忽略消息的重要依据。每个节点都具有一个属性，称为 currentTerm，每条消息都会携带发送者的 term。收到任何消息时，节点都会判断消息是否来自于以前的 term（`term < currentTerm`），如果是的话，直接无视。

term = 3 的节点收到消息 term=5，这意味着什么？意味着：

节点长期与其他节点隔离，以至于第四届，第五届选举时，投票没有通知他。

即便它贵为 term=3 的 leader，如今它也应该下台做 term=5 的 follower。

考虑网络异步化，我们重新整理 leader 选举的过程。

candidate 发起投票，消息中会携带它的 currentTerm。它可能出现 term 过低，被人无视的处境。此时 candidate 的本届选举相当于失败，它应当进行下一届选举（curentTerm++）。因为他坚信，leader 已经下台了。

follower 收到 candidate 的投票，如果投票的 term 足够大（比如 currentTerm=2，消息 term=3），follower 就会同意，并将自己的 currentTerm 设为 3。

![](http://og0xhkmh3.bkt.clouddn.com/raft/campaign.jpg)

（term = 3 的 candidate 发起投票，S2，S3，S5 同意）

candidate 收到过半的 follower 同意，则它当选 leader。

好了，我们再整理一个问题：candidate 发起投票，原因是 leader 下台，需要换届。而它又是如何得知 leader 是否下台的？

在平时，leader 会发送心跳（heartbeat）给所有 followers，告诉它们自己还活着。一旦 leader 失效，followers 没有收到心跳，那么它们自然会选举出新的 leader。

所以可能某个 candidate 发起投票，却被所有人无视，但只要他坚信 leader 已经下台（没收到任何来自 leader 的消息），那么他依然会发起新的选举。

到此这篇教程就结束了，Raft 论文中还有一些需要体会的内容，但对初识一致性协议的同学来说不那么重要。文章中有错误的地方欢迎指正，有不理解或者批评的地方也欢迎提出。

