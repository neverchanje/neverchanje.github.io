---
title: Redis Internals, sds
date: 2016-12-14
type: "post"
comments: false
tags: redis
---

sds 可以看成是一个针对短字符串专门优化内存的字符串类，也可以被当做是一块内存区（既可以用作 buffer，也可以用作 string，熟悉 golang 的都知道这在语义上是被区分的）。
sds 经过两个版本的发展，我们可以依次介绍一下这两个版本：

1.0

```c
struct sdshdr {
    int len;
    int free;
    char buf[];
};
```

这个版本的 sds 的特点：

- binary safe / 二进制安全，如果字符串中间有 ‘\0’，例如 ‘abc\0abc’，那 strlen 只会认 ‘\0’ 之前的内容。如果要修正这一点，就要将字符串长度 len 和字符串数据绑定在一起，类似 `std::string`。简单说，c string 在设计时为了放权给程序员，并没有把字符串长度跟字符串本身结合在一起，虽然几乎其他所有语言都是如此设计，但是 C/C++ 依然保持 zero-overhead。
- 为了兼容 libc 库的字符串函数（互操作性），sds 的末尾仍然以 ‘\0’ 结尾（这也算是一个牺牲）。sds 和 `char*` 等价（`typedef char* sds`），sds 指向结构体的 buf 区。这样 sds 和普通 `char*` 可以混用。
- sds 保证一定的易用性，主要是集中在它的自动扩容上。我们知道实现字符串类一般需要三个变量，长度，空间，数据，如果原有空间不够，则需要新开辟一段更大的空间，sds 隐藏了这部分细节（所以 sds 才被叫做 dynamic string），其实扩容算法是个问题，sds 的算法比较朴实，即一旦空间不够，就扩大到原来的两倍，直到 `SDS_MAX_PREALLOC`（1mb），也就是说一个 sds 最大只能有 1mb 数据。同时除了扩容功能，sds 也给了缩容功能（`sdsRemoveFreeSpace`）。
- 易用性还包括 sds 提供的其他字符串辅助函数，都比较便利。
- **性能**，任何性能提升都不是平白无故的。比如 sdslen 可以做到 O(1)，而 strlen 只能做到 O(n)，这是由于 sds 保存了长度信息。sdshr 的 layout 保证了 sds 是连续的一整块内存，len, free, buf 都是连续且紧凑地放置着。这在无形中优化了 cache line，同时也减少了内存碎片。
看完这么多其实我还是建议大家可以去亲自看看 sds 的 README.md，可以更多地知其所以然。

2.0

第二版的 sds 暂时缺少文档。
上一版的 sds 仍然有几个可以做 tradeoff 的地方：

1. 由于编译器自动会进行内存对齐，最终 sdshdr 的大小会是 8 的倍数（64 位机）。这在一定程度上虽然提升了性能，但是会浪费一定内存。

2. 对短字符串不友好，对极短字符串（短到两三字符，长到20字符），其实 strlen 不会有很大开销，而 len 和 free 的 overhead 就不可忽略了（相当于 8 个字符）。这点在一些平台的 std::string 得到了缓解（[C++ 工程实践(10)：再探std::string, 陈硕](http://www.cppblog.com/Solstice/archive/2012/03/17/168210.html)）。

问题1 的解决可以引入编译器选项 `__attribute__ ((__packed__))`，这样结构的大小和代码定义的就能完全相同，当然，这也会导致一定的性能下降。

问题2 比较难，redis 为不同长度的 sds 设计了不同的 header，原来的 header 是固定的，包含 8 bytes 的 len 和 free。现在的 header 能够根据不同的长度自适应地选择相应的 header。

```c
/* Note: sdshdr5 is never used, we just access the flags byte directly.
 * However is here to document the layout of type 5 SDS strings. */
struct __attribute__ ((__packed__)) sdshdr5 {
    unsigned char flags; /* 3 lsb of type, and 5 msb of string length */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t len; /* used */
    uint8_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr16 {
    uint16_t len; /* used */
    uint16_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr32 {
    uint32_t len; /* used */
    uint32_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr64 {
    uint64_t len; /* used */
    uint64_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
```

这里拿 sdshdr5 简单举个例子，原来的 header 需要 8 bytes，现在一个数据长度小于等于 `1<<5=32` bytes 的数据，只需要 1 byte 的 overhead。

sds 在引入了自适应选择 header 的策略之后，主要修改了原有的 sds 扩容和缩容的代码，其他基本保持不变。尽管如此，sds2.0 无法与 1.0 版本保证二进制兼容（因为 sds 的内存布局变了）。除此之外，API 没有发生任何变化（可以用 sdiff 确认），我认为应该可以平滑迁移。

每当 sds 扩容缩容的时候，都会根据新的数据长度，选择合适的 header，其实这点带来的开销不大，因为本身扩容缩容就是需要 O(n) 的。

除了我的博客之外，我推荐几篇还不错的文章：

- [zhangxiaolei: Redis内部数据结构详解(2)——sds](http://zhangtielei.com/posts/blog-redis-sds.html)

