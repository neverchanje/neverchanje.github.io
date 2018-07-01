---
title: baidu sofa-pbrpc 源码阅读：TimeoutManager
date: 2017-2-05
type: "post"
comments: false
tags: rpc
---

## Design

TimeoutManager 的设计思路是这样的：我们需要注册 event 到 TimeoutManager，每个 event 都有一个 timeout，一旦时间超过 timeout，事件就会发生。
每个事件其实就是一个 callback，事件发生时，执行 callback。

具体实现也很简单：一个后台线程（TimerWorker）负责 tick，每 10ms tick 一次，每次 tick 都遍历所有的 timer events，检查哪些 events 超期，如果超期就执行 callback。

这种设计从上层看起来就好像事件都一个个精准地按时发生了，其实不然，每个事件发生的时间都不是严格的 now() + timeout，而是根据 tick 的时间粒度，会有一定的误差。不过这通常是无所谓的。

其实这种设计已经非常经典了。redis ae，以前看到 libevent 也是这么实现（不确定）。TimeoutManager 是一个通用型的设计，如果用户只做 RPC，那这个类对于用户是隐藏的，但用户可以把它作为工具类使用，实用性很强。

## Implementation

一个可以讨论的点是 events 要用什么方式存储。

我们可以用 std::vector，在检查哪些 events 超期的时候，理论开销 O(n)。不过因为 timer events 一般不多，所以实际性能可以很快。这是 redis ae 的做法。也可以用 std::set，用一个二分查询找到超期的 events。sofa-pbrpc 用的就是 set。不过 set 如果按 expiration time 排序，那么可能有多个事件的 expiration time 相同。所以可以用 std::multiset。但是我们还要避免事件被重复插入，所以需要一个 unique id 来判断事件是否已经在 set 里头了，如果某事件已经在 set 里，则插入失败。所以现在需求定下来：我们需要一个既能保证 id 唯一，又能按照 expiration time 排序的数据结构。

```cpp
std::set<Id> id_set_;
std::multiset<Time> expiration_time_set_;
```

我们可以像上面那样。当然现在有更好的方法了：

```cpp
typedef boost::multi_index_container<
    Event,
    boost::multi_index::indexed_by<
        boost::multi_index::ordered_unique<boost::multi_index::member<
        Event, Id, &Event::id
        > >,
    boost::multi_index::ordered_non_unique<boost::multi_index::member<
        Event, int64, &Event::expiration
        > >
    >
> Set;
```

## Use Case

RpcServer 使用 TimeoutManager 作为后台定时事件发生器，实现几个功能：

## Keep Alive

显然 KeepAlive 是需要定时器的，一条 stream 如果超过时间没有读写，就应该清理掉。

## Flow Control

我以前没有设计过，更没有实现过 flow control，所以对这方面经验很薄弱。

sofa-pbrpc 按照时间片分配流量，用户指定每秒允许的总流量 x MB/s，sofa 会固定每 100ms 的流量是 x/10 MB/s。

```c
// Network throughput limit.
// The network bandwidth is shared by all connections:
// * busy connections get more bandwidth.
// * the total bandwidth of all connections will not exceed the limit.
int max_throughput_in;       // max network in throughput for all connections.
                             // in MB/s, should >= -1, -1 means no limit, default -1.
int max_throughput_out;      // max network out throughput for all connections.
                             // in MB/s, should >= -1, -1 means no limit, default -1.
```

## RPC Timeout

@qinzuoyan（sofa 作者） 告诉我，**为什么我们需要删除那些已经加到 TimeoutManager 的事件（TimeoutManager::erase），为什么每个 Event 需要一个 Id**。按道理定时事件加入 TimeoutManager 之后就应该放手不管，就好像我们不应该删除已创建的线程一样。

真实的场景是这样的：RPC 调用需要有 timeout，如果 RPC 超时，则进行错误处理。所以在 client 发起一个 RPC 时，我们注册一个定时任务到 TimeoutManager，如果超时，则进行错误处理。但大部分时候都不会超时，多数情况 RPC 都会成功，我们在成功之后，需要把定时事件从 TimeoutManager 中移除。否则假设每秒 10w 的请求量，定时算在 20s，那 TimeoutManager 会堆积百万量级的定时事件。

上面我们说到 EventId 用来避免事件重复插入，其实这是一个伪需求。真正的原因是我们要拿 EventId ，才能删掉一些无用的定时事件。

