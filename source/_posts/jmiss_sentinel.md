---
title: 从 JIMDB sentinel 到 Health Check
date: 2017-1-23
type: "post"
comments: false
tags: redis
---

## Intro to JIMDB

最近看到京东基础平台团队有人把设计放到微信公众号上，我就想以一个无知者的角度，评判一下 JIMDB sentinel 的设计。读者在读这篇文章之前，可以先看看公众号里的内容。

JIMDB 是一个基于 docker 的 redis 集群管理平台，功能相对是比较丰富的，比 redis cluster 要多了一些功能。比如 redis failover 时 docker 资源分配，监控数据采集。公开的介绍文章总是喜欢强调：京东拿它来管理数十万 Redis 实例 —— 其实明白人都知道这不代表技术就一定多牛逼，多稳定 —— 当然它肯定有其优秀之处的。

## Sentinel

我们看看他们怎么描述这个架构的：

> 多个探测实例同时对同一个JIMDB实例进行探测，只要一个探测实例检测到服务端实例是存活的，那么该实例就认为是存活状态，当没有人反馈其为存活状态，且超过半数的探测实例认为该实例死亡时，则通知故障恢复程序进行主从切换…

这个探测实例指的就是 sentinel，熟悉 redis 的应该知道，在定位上，它和 redis 2.8 即已引入的 sentinel 没有半点区别。我并不清楚他们自己重写一个的理由，不过如果把 sentinel 的实现从纯 C 转为 golang，那需要的代码行数应该会少一些。我觉得 sentinel 是适合用 golang 来做的。当我们把一个分布式系统的逻辑从复杂的代码里解放出来，我们就更容易编写正确的代码，加之它对性能也不那么敏感，用 C 写真是太费力了。

我们可以猜到上述提到的 “故障恢复程序” 就是指的 failover 模块。redis sentinel 里 failover 模块和 sentinel 是紧耦合的，我认为应该解耦出来。将 failover 解耦出来的设计应该是这样的：我们不需要考虑 master 底下有哪些 slaves，我们把 master 和 slave 都统一以 “节点” 为概念管理，一旦 sentinel 检测到某个节点出错，就提醒 failover 模块，由 failover 模块决定接下来的步骤，如果该节点是 master，就选一个 slave 接替。failover 模块也需要负责向资源模块（resource manager）申请新资源。

我认为 redis sentinel 并不是一个优秀的设计（当然我们必须承认 antirez 在分布式系统的经验，也必须承认 redis 在单机下是非常优秀的产品）。JIMDB 团队一开始有考虑过用 zookeeper 来代替 sentinel，然而他们认为 zookeeper 做健康监控太重，zk 很可能无法撑住如此多的 redis 实例。不过 zookeeper 可以用来做 sentinel 之间的 leader 选举。

> 在故障检测和故障切换的方案中，比较容易想到的方案就是引入zookeeper，通过zookeeper的临时节点探测不存活的服务，但是由于需要修改服务端代码、不方便跨机房部署、watch数目和连接数过多有性能问题等原因最终没有被采用…

我们可以参考 kubernetes 的设计，理论上一台物理机应该可以有十几台 redis 实例（这里我们假设每个 redis 实例占用一个 cpu core，当然部署上百台也是可以的）。我们可以让 sentinel 负责对一台物理机发 ping，物理机上开一个 agent 进程，负责监控机器上所有 redis 实例的健康状态，比如进程是否 hang 住。如果 agent 无法响应 pong，那说明物理机上一定是多数 redis 实例都停滞了（可能是网络问题，cpu 调度问题，io 调度问题）；如果一个 redis 实例挂了（比如某些 fatal error），agent 发现后就能返回错误状态给 sentinel。这样 sentinel 只需要维护 (number of machines = number of redis / 1x) 条长连接，相对会轻松很多，这时候就可以用 zk 做健康监控了。

![sentinel with pod](http://og0xhkmh3.bkt.clouddn.com/redis/sentinel_with_pod.PNG)

我们知道 sentinel 检测到 SDOWN 的时候会询问其他 sentinels，如果过半的 sentinels 也认为 SDOWN，则结果为 ODOWN，即客观下线。其实这带来一个 availability 的问题，假如 5 个 sentinel 集群统一管理上百（比较少了）个 redis 实例，假设放在同一机房，3 个 sentinel 出现网络分区，它们就会误报错误，认为所有的 redis 实例全部 ODOWN。按照原有逻辑，它们会 failover 所有的节点。sentinel 本质上既不是 cp 系统，也不是 ap 系统（观点来自 Jepson）。一个非高可用方案去做集群健康检查，我认为不是那么靠谱，这是任何集中式健康检查系统的通病。整个集群的可用性依赖于 sentinel 的可用性，这很可怕。说实话，心跳检测并没有那么的 critical，因为心跳服务不可用导致所有 redis 不可用，我认为是不可容忍的。

就上面关于 sentinel 误报错误的问题，我们可以加上一个逻辑：如果过多的 redis 实例出现 ODOWN，就不做 failover。当然这种方案靠不靠谱，读者可以自己考虑。

一个 sentinel 发现 master 失效后，它收集投票，过半则 ODOWN。这步没有问题，但在确认 ODOWN 之后，它需要将 ODOWN 日志分发给其他 sentinels，这步会带来一致性问题。我们可以用 zk 来做这次日志分发。

我认为 sentinel 需要一个严格正确的一致性方案，如果不行，那就干脆牺牲一致性。Jepson 尝试用 TLA 证明 sentinel 的正确性，然而他认为整个方案过于复杂，难以证明（不过 Jepson 在博文里一些观点有点问题）。

## Update

上面关于 sentinel 因为 CP 机制而失去高可用的结论，其实存在误导，读者有兴趣可以接着往下看。

# Health Check

## zookeeper

关于 zk 连接过多会带来性能问题，准确说是直接利用 zk 实现 health check 的方法对性能要求比较高。要实现 health check，我们可以让服务注册一个 ttl 节点到 zk，每隔一个心跳周期内，节点必须更新 ttl，否则节点超期，服务失效。

我们可以想到，每次心跳都要写日志到磁盘，更新一次 ttl，然后 commit 到 quorum。我暂时没有具体看过 zk/etcd 的 lease 实现，不过我认为写磁盘是不可避免的。所以简单地说，这种方案是不可容忍的。

当然读者可以辩解，现在机器性能越来越高，zk 还是能够支撑一定规模的集群的。

然而谁会不计成本的，专门用一个 zk 集群来做心跳呢。

## SWIM

上面我们说到集中式健康检查存在问题，而事实上，[SWIM](http://www.cs.cornell.edu/~asdas/research/dsn02-swim.pdf) 很早就提出了使用 Gossip 协议做非集中式，点对点的心跳检测，极大程度降低了误报率，提高 failure dection 的可靠性。开源实现对应 [hashicorp/serf](https://github.com/hashicorp/serf)。这是 AP 模型，虽然我不知道是否有企业采用，并且使用 Gossip 似乎会带来运维部署，日志追踪上的问题，但整个方案依然非常吸引我。

## Baidu/Galaxy

在 COISF 上，baidu galaxy 团队分享过他们在服务探活方面的[经验](http://www.10tiao.com/html/421/201702/2247484511/1.html)。与很多方案一样，其实本质上 galaxy 也是用类似 zk 的强一致集群（[iNexus](https://github.com/baidu/ins)）管理元数据的，其中就包括服务探活。

但与我们上面提到 zk 的服务探活方案不同，在 galaxy 上，真正的服务探活工作是 AppMaster 做的。一个集群可以有多个 AppMaster groups，在探活的过程中有任何元数据的交换，可以使用 ins 服务。我们知道，一般来讲只会在某一服务发生 failure 的时候才会需要 ins，而 failure 的概率并不高，这就缓解了 ins 的压力。

继续上面的讨论：AppMaster 真的会因为 ins 不满足 CAP 的 A，而不够高可用吗？

其实这么说是有失偏颇的。虽然 ins 本身不满足 A，但是我认为心跳服务应该是足够高可用的（单纯纸上谈兵的说）。心跳服务失效的概率是 服务失效概率，AppMaster 失效概率，ins 节点失效概率的乘积关系（具体不会算），这个概率具体能不能到高可用（99.999%），需要靠实践经验判断。galaxy 团队回答了我这个问题，他们说 galaxy 使用该方案至今还是比较稳定的。

另一方面，ins 的可用性可以通过错开机架，跨机房部署来获得一定的提升。如果可用性达到高可用的程度，理论上我们应该可以不需要在意这个问题。

## Conclusion

讲真的京东这篇文章关于 Sentinel 的介绍非常粗略，很多关键点都没有讲到，我个人比较反感这样的写作态度，当然或许他们有难言之隐呢，对吧。

## TODO

[ ] 我们之后看看 Codis 如何应对上述问题。

