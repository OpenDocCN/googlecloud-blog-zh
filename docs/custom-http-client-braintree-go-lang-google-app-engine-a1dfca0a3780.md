# 自定义 HTTP 客户端 Braintree Go-lang Google 应用程序引擎

> 原文：<https://medium.com/google-cloud/custom-http-client-braintree-go-lang-google-app-engine-a1dfca0a3780?source=collection_archive---------0----------------------->

![](img/f055d1cc3bce6ee45b6412176ea0bd46.png)

图片来源[https://blog.charmes.net/images/gopherswrench.jpg](https://blog.charmes.net/images/gopherswrench.jpg)

# 概观

Braintree 是最大的支付网关提供商之一，提供大量功能来执行支付操作。

Google App Engine 是一个完全托管的平台，它将基础设施完全抽象化，因此开发人员只需关注代码

> *布伦特里还没有为 Go-lang 提供官方 SDK。令人感激的是，有人为我们提供了完美的解决方案。*
> 
> 布伦特里围棋 SDK:【https://github.com/lionelbarrow/braintree-go 

# 问题

在过去的几个月里，当我在谷歌应用引擎机器上运行它时，无论是在 GAE 本地(dev_appserver **)** 还是当它被发布到 GAE 云时，我都很纠结。

SDK 抛出这个错误消息:

> "无法生成客户端令牌:Post[https://API . sandbox . braintreegateway . com:443/merchants/xxxxx/client _ token](https://api.sandbox.braintreegateway.com:443/merchants/xxxxx/client_token):http。默认传输和 http。默认客户端在应用程序引擎中不可用。参见[https://cloud.google.com/appengine/docs/go/urlfetch/](https://cloud.google.com/appengine/docs/go/urlfetch/)

有人和我有类似的问题:

> [https://github.com/lionelbarrow/braintree-go/issues/49](https://github.com/lionelbarrow/braintree-go/issues/49)

出现此问题是因为 App Engine 不允许我们使用原生 Go-lang http 客户端直接执行出站请求，所有内容都应通过 [urlfetch](https://godoc.org/google.golang.org/appengine/urlfetch) 。

# 解决办法

因此，我们需要附加自定义 http 客户端(GAE url 获取客户端)当我们创建布伦特里客户端。

这里有两个例子涵盖了这些问题:

*   创建布伦特里客户端(这里有一个问题) :

在上面的代码中，app engine 将阻止来自 SDK 的出站请求

*   解决方案，使用自定义 http 客户端创建 Braintree 客户端:

创建定制的 httpclient(第 2 行),然后将其传递给 braintree 客户端创建(第 8 行)

现在，它在 GAE 机器上运行得非常好。

仅此而已:)