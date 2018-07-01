---
title: Redis Internals, Other Features
date: 2017-1-24
type: "post"
comments: false
tags: redis
---

这篇文章主要是我自己做个整理笔记的，有点滥竽充数的意思，相关内容的详细描述很多都能 google 到更好的。

## HyperLogLog

HyperLogLog 是用来解决 distinct elements problem 的一种算法，比如计算访问某个 URL 的不同 IP 数，从而计算 DAU。
坦白说我曾经花了整整一天时间调研 HyperLogLog，不过还是没搞懂这套算法，最终放弃。

**参考**：

- [博主 CodingLabs 关于基数估计算法的专题博文](http://blog.codinglabs.org/tag.html#%E5%9F%BA%E6%95%B0%E4%BC%B0%E8%AE%A1)
- [antirez 关于 redis HyperLogLog 的介绍博文](http://antirez.com/news/75)

## Pub/Sub

实现消息队列的订阅发布功能。

我们可以认为，redis server 在内存中维护了如下的数据结构：

```cpp
map<robj*, list<client*>> pubsub_channels; // <channel> -> <list of clients>
```

**PUBLISH**
每发布一条消息，立刻对所有订阅该消息的 clients 发送该条消息。

**SUBSCRIBE**
订阅其实就是在 pubsub_channels[channel] 里加入当前的 client。等价于：

```cpp
pubsub_channels[channel].push_back(client);
```

订阅发布功能可以用在 sentinel 上，一旦 redis 实例出现了偏差，sentinel 就通知订阅的客户端。

## RedLock

RedLock 其实不是一个 feature，它只是利用 redis 来做分布式锁的一个算法，具体的实现交给社区，目前看应该没有一个经过线上考验的官方实现，再加上这个 proposal 也算是半个伪需求，哪怕 redis 在自己之上实现一个 raft，我也很难不去考虑更 full-featured 的 zk 和 etcd。

[Distributed locks with Redis](https://redis.io/topics/distlock)： 关于 Redlock 的官方文档。假设 5 Redis 实例，申请锁就向每个实例依次发起请求，其中有一个请求超时就不断重试，释放锁就向每个实例请求释放。在锁之上做 lease，如果申请锁的过程耗时超过 lease 时长，则当做申请锁失败。最终如果过半实例判决同意锁申请，就算申请成功。
这个算法可吐槽的地方太多，我都不知道从哪里吐槽起。我个人对 antirez 是十分尊敬的，我从他身上学到了很多做工程和做开源的经验。但是我们应该能意识到，分布式锁算法本质涉及到一致性问题，而 RedLock 其实只负责了 log replication，failover 阶段也只负责了 recovery（把锁的获取操作日志利用 aof 持久化，后续可以从日志里 recover 起来），而没有负责新节点的 catch up。不考虑 catch up，这个算法就是不完整的，一个不完整的算法是没有对错之分的。catch up 需要一个分布式递增计数器，这个算法里没有提供（id 用的是 UUID）。
RedLock 算法强制使用 lease（antirez 把这叫 auto-release，即锁经过一段时间就会自动释放），我们假设是 short lease，虽然在一个 lease 内的操作正确性不那么好保证，但是经过 lease 后锁会失效，一定程度上可以减轻错误的代价。不过这也说明 RedLock 作为一个分布式锁算法，本质是在用 correctness 做 tradeoff，增加了 performance。所以正如下面的博文作者 Martin 提到的，与其花 5 台 redis 做 RedLock，还不如直接用 2 台 redis 做主从切换。

[How to do distributed locking, Martin Kleppmann](http://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html)：这篇博文干货太多了。背景是这样的，antirez 提出 RedLock 算法，希望社区有人可以评估一下，然后博主有理有据地否定了这个算法，当然这是个一黑顶十粉的事情。
Martin 提出一个蛮有意思的问题，假设 client 申请到锁之后，lease 还没超时，于是 client 尝试对某个 shared resource 做写操作，如果此时写操作的过程中进程出现很长的停滞（比如 STW GC），此时 lease 超时，理论上写操作应该失效，然而我们没有任何的机制可以让写操作失效。如果不让它失效，旧的写操作会覆盖新的写操作。

![redlock gc block](http://og0xhkmh3.bkt.clouddn.com/redislock_unsafe-lock.png)
这个问题不是 RedLock 的锅，理论上任何使用 lease 策略的分布式锁都会有这个问题，比较麻烦。

[Is Redlock safe?](http://antirez.com/news/101)：这就是 antirez 应对 Martin 的批判写的博文。
坦白说我对分布式系统的时钟相关的知识储备比较薄弱。

参考：

[聊一聊分布式锁的设计](http://weizijun.cn/2016/03/17/%E8%81%8A%E4%B8%80%E8%81%8A%E5%88%86%E5%B8%83%E5%BC%8F%E9%94%81%E7%9A%84%E8%AE%BE%E8%AE%A1/) 这篇文章可能有一定误导性，我个人不是很喜欢，不过可以作为参考。

## Transaction

redis 的事务 官方文档。我经常忘了事务怎么写，所以在这里记一下：

```
> MULTI
OK
> INCR foo
QUEUED
> INCR bar
QUEUED
> EXEC
1) (integer) 1
2) (integer) 1
```

客户端发送 MULTI，输入一系列指令，输入 EXEC 执行，返回结果。一开始我以为 EXEC 之前的操作都可以交给 client 去做，等到 EXEC 再将事务所有命令发送给 server，减小 server 的负担。然而 redis 在这里的每一个命令都会与 server 交互，像 INCR foo，INCR bar 这些命令，都会经过 redis 的正确性检查，只有命令正确才会返回 QUEUED。
所以说明 redis 协议必须是有状态的。
这可能也是为了减轻 client 的负担，避免不同语言 clients 的实现参差不齐。

```
127.0.0.1:6379> multi
OK
127.0.0.1:6379> fuck
(error) ERR unknown command 'fuck'
127.0.0.1:6379> exec
(error) EXECABORT Transaction discarded because of previous errors.
```

## Authentication

redis 应该只支持 plain text 的 authentication。相关原因可以参考官方文档 [Redis Security](https://redis.io/topics/security)。
唯一一个考虑到安全性的地方，是 `server.c:time_independent_strcmp`，这里用来比较密码。如果用 strcmp，我们可以通过执行时间的长短，猜出密码。用 `time_independent_strcmp` 的目的是想让不管什么长度的密码，比较的耗时都一样，攻击者就猜不出密码。

我来说下用 strcmp 的时候怎么猜密码：攻击者猜一个密码，假设 n 个字符猜对了前 k 个，我们尝试猜第 k+1 个，如果猜对了，比较耗时应该更长。猜错了，耗时不变。这样我们就可以暴力把密码猜出来。

`time_independent_strcmp` 在实现上确实避免了这种攻击方式，不过显然的安全性还不够。可以参考 mongodb 在 SASL 框架之上加了 SCRAM-SHA1，还支持 TLS。当然，这些官方肯定比我们更清楚。

## TODO

之后我们有必要调研一下 [Jepsen: Redis](https://aphyr.com/posts/283-jepsen-redis) 这篇博文的内容。
未来我要看一下 etcd 的 lease 实现，RedLock 的问题在 etcd 中一定也会有。

