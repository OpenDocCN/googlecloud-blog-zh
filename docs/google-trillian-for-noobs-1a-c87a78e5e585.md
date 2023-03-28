# Google Trillian for Noobs (1a)

> 原文：<https://medium.com/google-cloud/google-trillian-for-noobs-1a-c87a78e5e585?source=collection_archive---------1----------------------->

## 失踪手册系列

上周，我记录了我所希望的最简单的崔莉恩人格。这是一个临时的帖子，因为我意识到我的示例中遗漏了一些重要的功能，一个包含证明:一些指定数据是透明日志的一部分的有效的无可争议的证据。

## 设置

您将需要我在之前的帖子中描述的数据库和 Trillian 服务器。

## 基本++人格

这次，请克隆`inclusion-proof`:

```
git clone \
--single-branch \
--branch=inclusion-proof \
[git@github.com](mailto:git@github.com):DazWilkin/simple-trillian-log-1.git
```

然后:

```
GO111MODULES=on \
**GOPROXY=**[**https://proxy.golang.org**](https://proxy.golang.org) \
go run github.com/DazWilkin/simple-trillian-log-1 \
--tlog_endpoint=:8090 \
--tlog_id=${LOGID}
```

> 非常有趣的是，Go 团队正在评估 Go 模块的模块镜像。镜像不仅打算为下载提供一个快速代理，而且它利用了`go.sum`的散列来提供包的可验证性。非常非常有趣的是，这个解是一个 Trillian 人格；-)

您应该会看到类似于以下内容的输出:

```
[main] Entered
[main] Establishing connection w/ Trillian Log Server [:8090]
[main] Creating new Trillian Log Client
[main] Creating Server using LogID [5201727448936460865]
[server] Creating
[main] Creating a 'Thing' and something 'Extra'
[thing:new] Creating: [2019-07-03T13:01:34-07:00] Thing
[extra:new] Creating: Extra
[main] Submitting it for inclusion in the Trillian Log
[main] Awaiting Inclusion (Proof) in the Trillian Log
[main] Retrieving it from the Trillian Log
[main:get] Entered
[server:get] Entered
[thing:marshal] Marshaling: [2019-07-03T13:01:34-07:00] Thing
[server:get] hash: 0800eb65...
[main:wait] Entered
[server:wait] Entered
[main:put] Entered
[server:put] Entered
[thing:marshal] Marshaling: [2019-07-03T13:01:34-07:00] Thing
[extra:marshal] Marshaling: Extra
[server:wait] Root hash: 68f7634b...
[main:get] Status:ok
[main:get] Done
[thing:marshal] Marshaling: [2019-07-03T13:01:34-07:00] Thing
**[main:wait]** rpc error: code = NotFound desc = No leaf found for hash: 0800eb65... in tree size 39
[main:wait] Status:
[main:wait] Sleeping
[main:put] Status:ok
[main:put] Done
[server:wait] Entered
[server:wait] Root hash: 6c7a5673...
[thing:marshal] Marshaling: [2019-07-03T13:01:34-07:00] Thing
[main] proof[0],hash[0] == 9dc9b7d5...
[main] proof[0],hash[1] == 384214f3...
[main] proof[0],hash[2] == 36e35370...
[main] proof[0],hash[3] == 880846d5...
**[main:wait]** Status:ok
[main:wait] Done
[main] Done
```

这段代码试图镜像日志的使用。在`main.go`有 3 个戈罗廷。其中之一——`put`——尝试向日志中异步添加一个`Thing`。另一个— `get` —尝试从日志中异步获取`Thing`。第三个— `wait` —尝试对添加到日志中的`Thing`执行包含证明。在`put`成功之前`get`和`wait`不能继续。更重要的是，`wait`直到能够计算包含证明才能成功。因此，在上面的日志输出中，虽然`get`第一次尝试成功，但是`wait`失败，休眠一秒钟，重试成功。

## Trillian 客户端

当我询问实现 inclusion proof 的细节时，Trillian 团队向我介绍了 Trillian Client ( `trillian/client`)包。我没有意识到它的存在:-)本示例中的代码使用机器生成的(来自 Trillian 的 protobufs) gRPC 客户端。然而，Trillian 客户端是在此之上的一个抽象，它还包括包含证明功能。在您的代码中，我建议您使用`trillian/client`而不是原始的 gRPC 客户端。

## 结论

我们现在有了一个客户端(`main.go`)角色，它与一个万亿日志交互来添加、检索和证明`Things`的存在。在下一个故事中，我们将把这个客户机变成一个服务器，使用 gRPC 公开它，并生成一个(测试)客户机来使用它。决赛中(？)贴，我计划(！)来尝试 Rust，并为这个角色生成一个 Rust 客户端。这实际上不是 Trillian，但希望它能帮助您了解如何使用该解决方案。

仅此而已！