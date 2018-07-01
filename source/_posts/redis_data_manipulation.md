---
title: Redis Internals, Data Manipulation Operations
date: 2016-12-19
type: "post"
comments: false
tags: redis
---

## List

两年前刚从区域赛回来，开始去读 redis 的源码，当时还是 3.0 时代，list 命令会根据不同的场景使用 adlist 和 ziplist。现在已经完全使用 quicklist 了。

我们可以回顾一下以前的方法：

- 当 list 较短的时候，使用 ziplist 进行存储，在 list 较长的时候，使用 adlist 存储。具体什么时候会从 ziplist 转到 adlist？redis 提供了 list-max-ziplist-entries 选项，默认值为 512。如果 ziplist 的元素的个数超过 512，就会从 ziplist 转到 adlist。触发这个转换的时机有：PUSH，LSET 操作。（`<=3.0`）
- 在删除节点的时候并不会尝试去从 adlist 转到 ziplist。（`<=3.0`）

显然，一旦一个 list 的生命在某时刻从 ziplist 转换成 adlist，它就再也回不到 ziplist 状态了。这对一个长期运行的程序来讲是不利的，因为可能这个 list 只是遇到了热点才变大，在它慢慢变冷之后应该回到 ziplist。

可以联想到，说不定会有一些人为了避免 list 永远退化为 adlist，所以选择将一个 adlist 量级的 list sharding 到多个 ziplist 量级的 lists 上，一旦一个 ziplist 满了，就按某种策略进行拆分。这个想法其实就是 quicklist 的思想。

在 3.2 版本的推进过程中，redis 引入了 quicklist，据说在一些场景可以提升 10x 性能（[3.2 release-notes](https://github.com/antirez/redis/blob/3.2/00-RELEASENOTES#L1824)）。quicklist 的思想是将 adlist 和 ziplist 结合，每个 node 是一个 ziplist，用 doubly linked list 的方式串起来。讲到这里我们基本就可以确定这是个 great idea 了。这里我们不讲具体实现， [Redis内部数据结构详解(5)——quicklist，zhangtielei](http://zhangtielei.com/posts/blog-redis-quicklist.html) 已经讲的足够好了。

## Set

set 操作是 unordered 的，但是保证唯一性，底层有两种实现，dict 和 intset。对这两个数据结构的简单了解可以看我的博客

## dict

这里可以吐槽，dict 居然一点不改就直接被拿来做 set。redis 是这么做的：

```
## SADD key member
redis> SADD myset "Hello"
(integer) 1
redis> SADD myset "World"
(integer) 1
redis> SADD myset "World"
(integer) 0
redis> SMEMBERS myset
1) "Hello"
2) "World"
```

加一个 member 到 myset，底层对应是加一个 dictEntry 到 dict。dictEntry 是一个 key value 组合，这里的 key 是 member，value 是 NULL。这里也是不合理的，之后看看提一个 issue。

```c
if (subject->encoding == OBJ_ENCODING_HT) {
    dict *ht = subject->ptr;
    dictEntry *de = dictAddRaw(ht,value,NULL);
    if (de) {
        dictSetKey(ht,de,sdsdup(value));
        dictSetVal(ht,de,NULL);
        return 1;
    }
}
```

## intset

如果集合里都是整数，就使用 intset。intset 是有序的，使用二分搜索，search 效率较高（O(logn)），所以即使 set 不要求有序性，使用 intset 也是合理的。
不过由于 intset 使用数组存储，集合一旦大了效率就会降低。redis 提供了 set-max-intset-entries 选项，默认值为 512，如果 intset 所的元素个数大于 512，就将底层存储从 intset 转到 dict。
同时，一旦往一个 intset 加入字符串类型的 member，intset 也会自动转换为 dict。

讲真的，这个实现还不太理想，希望有人把握这个机会写一个 quickset 出来。

## Hash Table

redis 的哈希表实现是这样的，如果 entries 个数小于 hash-max-ziplist-value，就使用 ziplist 来存。
这个实现比较奇葩，理论上这不是哈希表，构造如下：

```
|--field1--|--value1--|--field2--|--value2--|--...--|
```

这就导致此时的 HGET 操作是 O(n) 的，每次都是一个遍历去查。不过由于 ziplist 本身就对 cacheline 有优化，所以在元素数量少的情况下可能是可以容忍的。

entries 个数大于 `hash-max-ziplist-value` 的时候使用 dict 来存，这就不说了。

