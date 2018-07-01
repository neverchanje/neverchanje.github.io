---
title: Redis Internals, SET key value
date: 2016-12-16
type: "post"
comments: false
tags: redis
---


## SET key value

虽然 value 的类型固定是 string（可以使用 TYPE key 查看，对应的是 redisObject::type），但是 value 可能有多种 encodings。

```
> SET key 10000
OK
> TYPE key
"string"
> STRLEN key
(integer) 5
################################################
虽然我们看到的类型是 string，其实底层用 long 存储。
这里有点欺骗用户的意思，不过想想 redis 协议应该是
想隐藏底层存储的细节。
```

## Encodings

（参考 `object.c:tryObjectEncoding`）

熟悉序列化的肯定知道，8 字节 long 能存 20 个字符长度的整型数（LONG_MAX=9223372036854775807，算上负号就是20字符），对于大整数应该选用 long 而非 string。所以当 value 是一个整数，redis 尝试将其存储为 long。

这种方案显然对于小整数不友好，比如 long 存储整数 10 至少需要 8 bytes（64位机），而 string 只需要 3 bytes（使用 sds）。对于这个问题的一种优化方案是，对于小整数，我们仍然使用 sds 存储，大整数用 long 存储。redis 的优化方案更优秀。redis 将 0-10000 的整数放在公有常量池。这点在大量存储整数的场景，应该能够极大提高系统的最大负载。

## Embeded String（new feature in 3.0）

我们知道，value 的值存在 redisObject::ptr。

```c
typedef struct redisObject {
    unsigned type:4;
    unsigned encoding:4;
    unsigned lru:LRU_BITS; /* LRU time (relative to server.lruclock) or
                            * LFU data (least significant 8 bits frequency
                            * and most significant 16 bits decreas time). */
    int refcount;
    void *ptr;
} robj;
```

## embeded string 的 cacheline 优化

我们知道，sds 有一点很特别，就是它的内存布局是连续的

```
|----sdshdr----|----data----|

```

这意味着，每次去对 data 进行扩容，不光 data 要搬家，sdshdr 也要搬家。sdshdr 不大（有兴趣可以阅读 redis sds 的实现），所以搬家压力不大。一般的策略上，sdshdr 和 data 应该分两块内存区，这样 data 搬家的时候，sdshdr 不会搬。而 sds 设计成连续的，有各方面原因，其中一方面是可以优化 cache line，减少 cache misses。

我们这里讲清楚 cache line 优化的问题：在 cpu 访问 sdshdr 的时候，data 区前面的一部分也会被一起读入 cacheline。cacheline 的大小通常是 64 bytes，假如我们想把 sds 全部放到 cacheline，除去 sdshdr 的 3 bytes（sdshdr8）和 null terminator ‘\0’，还有 60 bytes 的数据会读入 cacheline。当然，如果 sds 的数据量远大于 60 bytes，那其实 cacheline 的优化就可以忽略不计。

现在回头看 redisObject，它的 header 有 16 bytes，64bytes 的 cacheline 除去 robj header，有 48 bytes。这个大小可以容纳 sdshdr8 的 sds，扣掉 sdshdr8 占用的 3 bytes，还有 sds 的末尾 ‘\0’，实际能够读入 cacheline 的数据量可以在 44 bytes。

所以 redis 在这里做了这么一个优化：如果数据长度小于等于 44 bytes（OBJ_ENCODING_EMBSTR_SIZE_LIMIT），就用 sds 的方法，将 robj header 和实际数据连在一起放置。

## 公有常量池

前面说 SET key value 存储 value 时，如果 value 是一个小于 10000 的整数，则 redis 将 key 指向公有常量池里值为 value 的 robj。这样可以有很多个 keys 共享一个 value。这个思想可以放在很多地方。

```c
struct sharedObjectsStruct {
    robj *crlf, *ok, *err, *emptybulk, *czero, *cone, *cnegone, *pong, *space,
    *colon, *nullbulk, *nullmultibulk, *queued,
    *emptymultibulk, *wrongtypeerr, *nokeyerr, *syntaxerr, *sameobjecterr,
    *outofrangeerr, *noscripterr, *loadingerr, *slowscripterr, *bgsaveerr,
    *masterdownerr, *roslaveerr, *execaborterr, *noautherr, *noreplicaserr,
    *busykeyerr, *oomerr, *plus, *messagebulk, *pmessagebulk, *subscribebulk,
    *unsubscribebulk, *psubscribebulk, *punsubscribebulk, *del, *unlink,
    *rpop, *lpop, *lpush, *emptyscan,
    *select[PROTO_SHARED_SELECT_CMDS],
    *integers[OBJ_SHARED_INTEGERS],
    *mbulkhdr[OBJ_SHARED_BULKHDR_LEN], /* "*<value>\r\n" */
    *bulkhdr[OBJ_SHARED_BULKHDR_LEN];  /* "$<value>\r\n" */
    sds minstring, maxstring;
};
```
