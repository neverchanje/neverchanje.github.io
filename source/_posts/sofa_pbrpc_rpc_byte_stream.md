---
title: baidu sofa-pbrpc 源码阅读：RpcByteStream
date: 2016-12-31
type: "post"
comments: false
tags: rpc
---

## Boost.Asio

由于 sofa 建立在 asio 之上，所以读者有必要去了解一下 Boost.Asio 库。这是一个 Proactor 模型 的 io 库。

参考资料：

- [Boost.Asio C++ 网络编程](https://www.gitbook.com/book/mmoaay/boost-asio-cpp-network-programming-chinese/details)

## RpcByteStream

按照 sofa 的设计，RpcByteStream 对应传输的最底层，负责 socket 通信。

![sofa-pbrpc stream](http://og0xhkmh3.bkt.clouddn.com/sofa_rpc_stream.png)

RpcByteStream 是一个虚类，子类有 RpcServerMessageStream，RpcMessageStream。留给子类实现的接口有：

- on_closed
- on_connected
- on_read_some
- on_write_some
- trigger_send
- trigger_receive

## 实现细节

每个 RpcByteStream 有 128 bytes 的 buffer 用来存 error message，这个未来可能可以优化。
析构的时候直接调用 `basic_socket::close()` 关闭 socket，而不是用 `RpcByteStream::close`。
`RpcByteStream::close(const std::string& reason)` 用于连接出错时关闭 socket（比如异步连接超时，或是 TCP-NoDelay 设置失败）。
底层实现用 `basic_socket::shutdown(basic_socket::shutdown_both)`，关闭双方的 socket。close 时输出 reason。

## 异步连接

异步连接超时功能用 `basic_socket::async_connect` + `deadline_timer` 实现。async_connect 的本质其实也是用 io_service 的 post 打入 eventloop，不阻塞，立刻返回。然后用 deadline_timer 进行等待（等待同样是异步的，用 `boost::asio::basic_deadline_timer::async_wait`，超时则执行回调函数），超过 connect_timeout（`deadline_timer::expire_from_now`） 则将会直接关闭 socket（`RpcByteStream::on_connect_timeout`）。
sofa 只提供了异步连接方式，当然 asio 本身是提供了同步连接的。
唯一一个会调用 `RpcByteStream::async_connect` 的地方是 `RpcClientImpl::FindOrCreateStream`，所以异步连接针对的是 client 主动连接 server 的时候。
场景是这样的：在一次 RPC 调用中（`RpcClientImpl::CallMethod`），client 发送请求给 server，对每一个 server 会建立一条 stream（`RpcClientImpl::FindOrCreateStream`）。创建一条 stream 的同时，会进行一次异步连接。异步的做法能够降低 client 的 delay time，因为在等待连接建立的过程中，客户端还可以顺便构造请求报文。除了异步的方式之外，keepalive 等机制也能省去等待连接建立的开销，这是后话。

## 连接建立，`RpcByteStream::set_socket_connected`

我们描述一下这个函数的使用场景：server 监听 client 发送来的 tcp 连接请求（`RpcListener::start_listen`，底层利用 `asio::tcp::acceptor`），如果 tcp 连接建立成功（`RpcListener::on_accept`），则将状态设置为 connected，即 set_socket_connected 。

## 总览
我们可以看到 RpcByteStream 在建立连接的过程中，维护了一个小的状态机。

```cpp
enum {
    STATUS_INIT       = 0,
    STATUS_CONNECTING = 1,
    STATUS_CONNECTED  = 2,
    STATUS_CLOSED     = 3,
};
```

RpcByteStream 把状态机的状态转移对子类完全隐藏。

![rpc_byte_stream_state_machine](http://og0xhkmh3.bkt.clouddn.com/rpc_byte_stream_state_machine.png)

这里我们就可以完整地理清思路，Client 异步发起连接（Connecting），并在等待的过程中准备 RPC 请求报文，这中间如果发生任何的错误，包括等待时间过长，则关闭双方的 socket（Closed）。如果服务端 accept，则连接建立成功，两方都进入 Connected 状态。如果服务端在 accept 的过程中出现任何问题，则关闭双方 socket（Closed）。

## Nagle 算法

在连接建立的过程中（CONECTING -> CONNECTED），双方都会进行 Nagle 算法的配置。
编译的时候可以指定 SOFA_PBRPC_TCP_NO_DELAY 选项，默认为 true，即不使用 Nagle 算法（应该是因为 RPC 场景通常重在快速响应？）。Nagle 算法即把数据包堆积在一起，堆到 MSS（ Maximum Segment Size） 再发送，提升吞吐量。
代码里写的挺清楚：

```cpp
// Disabling the Nagle algorithm would cause these affacts:
//   * decrease delay time (positive affact)
//   * decrease the qps (negative affact)
```

## Keep-Alive

sofa-pbrpc 可以配置 keepalive 机制。在 socket 断开之后，server 并不会直接关闭 stream，而是保留，并且每个 tick 去清理一次那些长时间没有读写的 stream。

```cpp
// Get the last time ticks for read or write.
int64 last_rw_ticks() const
{
    return _last_rw_ticks;
}
```

last_rw_ticks 记录上次读写的时间，如果 now - last_rw_ticks >= keep_alive_ticks，就关闭该 stream。

留给子类的接口
我们有必要提一下 RpcByteStream 留给子类的几个接口会在哪些地方被调用，这样可以方便未来我们介绍这些子类。

- `on_closed`：因为出错而调用 RpcByteStream::close 时用到。
- `on_connected`：只有在 client 进行异步连接的时候会用到，不过 RpcServerMessageStream 仍然实现了 on_connected（不然会编译出错）。这个设计足够，不过不够漂亮。
- `on_read_some`：RpcByteStream 提供一个 protected 的 async_read_some 方法，在异步读到数据的时候调用 on_read_some。
- `on_write_some`：与上面的 on_read_some 差不多。
- `trigger_send` 和 `trigger_receive`：这两个函数在 client 和 server 连接建立时，双方都会调用。

