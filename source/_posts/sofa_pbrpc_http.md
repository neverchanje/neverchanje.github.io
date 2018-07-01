---
title: baidu sofa-pbrpc 源码阅读：HTTP
date: 2017-1-08
type: "post"
comments: false
tags: rpc
---


原生支持 http 是 sofa 的一个重要需求。官方文档对 HTTP 支持有较完善的描述。

## 简单运行

在了解 HTTP 实现之前，我们可以先了解一下 sofa 的 server 端是如何运行起来的，这里参考源码中的 sample/echo。

![sofa-server-startup](http://og0xhkmh3.bkt.clouddn.com/sofa_server_startup.png)

启动 RpcServer 之后，我们即可以通过 HTTP 发送请求。

```
# 使用 httpie 发送 request
http -j -v http://localhost:12321/sofa.pbrpc.test.EchoServer.Echo message="Hello, world"
# request 内容
POST /sofa.pbrpc.test.EchoServer.Echo HTTP/1.1
Accept: application/json
Accept-Encoding: gzip, deflate, compress
Content-Length: 27
Content-Type: application/json; charset=utf-8
Host: localhost:12321
User-Agent: HTTPie/0.8.0
{
    "message": "Hello, world"
}
# response 内容
HTTP/1.1 200 OK
Access-Control-Allow-Origin: *
Content-Length: 40
Content-Type: application/json
{
    "message": "echo message: Hello, world"
}
```

## HTTP Or Binary

仔细想想会觉得莫名其妙，上面的流程图里，我们没有在任何地方指定使用 HTTP 作为请求的方法，然而 sofa 却可以判断我们使用 HTTP，并且返回 HTTP response。这是否意味着消息传输只能用 HTTP？

其实并非如此，sofa 还支持 Binary Message 的消息传输方式，具体格式可以参考官方文档。Binary Request 与 HTTP Request 类似，它也具有 header 部分和 body 部分。Binary Message 相比 HTTP message 最大的不同在于，整个信息使用的是尽可能紧凑高效的序列化二进制，而非简单可读的文本。

Binary Request 的 header 部分前四字节为 magic string，而 HTTP Request 的 header 部分前四字节为 method 字段（sofa 只支持 POST 和 GET）。所以我们可以通过 request message 的前四个字节，判断请求是 Binary Request 还是 HTTP Request。具体流程如下：

服务端接收请求，通过确认 magic string（`HTTPRpcRequestParser::CheckMagicString`），可以知道该请求是一个 HttpRpcRequest，随后即进行处理（`HTTPRpcRequest::ProcessRequest`）。

## Servlet：用 sofa 做 web 服务器

通过上图我们可以看到，除了利用 protobuf 做 RPC 以外，sofa 还能直接拿来写 web 服务器。我们可以在某个 URL 子目录（path）下注册回调函数（`RpcServerImpl::RegisterWebServlet`），这样就可以在访问某个 URL 的时候执行对应操作。

```
http http://localhost:12321/hello
# response 内容
HTTP/1.1 200 OK
Access-Control-Allow-Origin: *
Content-Length: 41
Content-Type: text/html; charset=UTF-8
<h1>Hello from sofa-pbrpc web server</h1>
```

Servlet 就是这里的回调函数。Server 接收 HTTPRequest，经过 Servlet 的处理，返回 HTTPResponse：

```cpp
typedef ExtClosure<bool(const HTTPRequest&, HTTPResponse&)>* Servlet;
```

上面的例子里，我们的 Servlet 接收到任何 request，都会返回 `<h1>Hello from sofa-pbrpc web server</h1>`。

其实非要拿 sofa 做 web 服务器就太牵强了，我们更应该用其他更广泛使用的开源 web 服务器。sofa 实现 servlet 的一个重要理由其实是可以[做监控页面（建议读者自己尝试一下）](https://github.com/baidu/sofa-pbrpc/wiki/%E9%AB%98%E7%BA%A7%E4%BD%BF%E7%94%A8#web%E7%9B%91%E6%8E%A7)。每个内置的监控页面都注册了对应的 Servlet。本质上用户也可以针对自己的使用，注册相应的 web 监控页面。

能想出这个功能也是很可爱。因为正常情况下都是返回一个 JSON，谁会自己用 C++ 写个 html 出来。不过当然这也启发我们联想，可能我们有一个特殊的 RPC server，它的监控信息需要用前端渲染，这时候用 C++ 写 html 是绝对不行的。我们可以嵌入 JS，可以加 css。

## 请求使用 JSON 还是序列化的 protobuf

HTTP 的 request body 默认以 JSON 传输。还是用 [sample/echo](https://github.com/baidu/sofa-pbrpc/tree/master/sample/echo) 中的 EchoService 作为例子。

```
# 使用 POST
curl -d '{"message":"Hello, world!"}' http://localhost:12321/sofa.pbrpc.test.EchoServer.Echo
# 使用 GET
curl http://localhost:12321/sofa.pbrpc.test.EchoServer.Echo?request=%7B%22message%22%3A%22Hello%2C%20world%21%22%7D
```

我们可以用 POST 和 GET 发送 JSON 请求。JSON 的格式必须符合 proto 规定的 schema。

```proto
message EchoRequest {
    required string message = 1;
}
```

所以我们发送的 JSON 一定要有且只有 message 这一项。Server 会对 JSON 的格式进行检验（`pbjson.cc:jsonobject2pb`）。

![sofa process http request](http://og0xhkmh3.bkt.clouddn.com/sofa_process_http_request.png)

HTTP 同样可以直接发送序列化的 protobuf 作为 request body。但是必须使用 sofa 自定义的 POST_PB 方法。

## ServiceBoard 和 MethodBoard

一个 RPC Server 一般注册了多个 services。

