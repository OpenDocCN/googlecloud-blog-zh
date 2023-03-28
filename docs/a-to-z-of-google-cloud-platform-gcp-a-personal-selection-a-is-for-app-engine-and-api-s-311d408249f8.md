# 谷歌云平台(GCP)的 A 到 Z 个人选择— A 代表应用引擎和 API

> 原文：<https://medium.com/google-cloud/a-to-z-of-google-cloud-platform-gcp-a-personal-selection-a-is-for-app-engine-and-api-s-311d408249f8?source=collection_archive---------1----------------------->

App Engine 是 GCP 的 PaaS。

App Engine 有很多很酷的东西，我在这里只简单介绍其中一个**特性**

我觉得“特性”反映了成为“最简单”云的哲学，因为它只提供你必须自己编码或喂水的功能。

# **特色**

这些大多是在很多用例中频繁使用的服务的 API。有些在 GA 中，有些在 beta 或 alpha 中。

考虑基本的图像操作，如调整大小和重新格式化。这是一个常见的需求，所以 App engine 提供了一个[图像 API](https://cloud.google.com/appengine/features/#images) ，这让它变得非常简单。图像服务可以直接从应用程序接受图像数据，也可以使用从谷歌云存储中检索的数据。

缓存是一个非常常见的需求，因此 App Engine 提供了 [memcache](https://cloud.google.com/appengine/features/#memcache) 这些只是两个真正有用的特性的例子，作为开发人员，您不必花时间编写功能或亲自照看服务。基本上你只是消耗它。

在撰写本文时，App Engine 提供了一份令人印象深刻的[特性列表](https://cloud.google.com/appengine/features/)。

使用功能有助于您快速提供基本功能，与 App Engine 的集成使开始使用功能变得非常简单。

当我第一次开始使用 App Engine 时，我发现它的特性与 API 管理系统有很多共同之处，只是没有时髦的标签。(有点像 Devops 并不是什么新东西，但是一旦它有了一个好的标签..)

要理解为什么我会如此大胆地说这句话，你需要理解我所说的 API 管理是什么意思:

*   一种允许发现 API 的机制
*   易于使用的有文档记录的 API
*   网关保护对 API 及其消费者之间流量的访问，并提供使用指标
*   生命周期管理，用于管理 API 的设计、开发、部署、版本控制和淘汰流程

我认为 App Engine 特性符合这个标准。

**允许通过控制台发现 API** 的机制。您可以从[特性表](https://cloud.google.com/appengine/features/)开始，向下钻取特性。如果是客户端消费，那么这就是应用程序和客户端之间的契约(参见端点)

**一个易于使用的文档化 API**—API 针对支持的语言进行了文档化: [App Engine Python API 的](https://cloud.google.com/appengine/docs/python/refdocs/)， [Google App Engine Java API 的](https://cloud.google.com/appengine/docs/java/javadoc/)， [PHP App Engine API 的](https://cloud.google.com/appengine/docs/php/refdocs/)

每个功能都有一个支持语言的专用文档页面，显示如何使用服务并列出配额，例如[图片 Python API 概述](https://cloud.google.com/appengine/docs/python/images/)

**保护 API 与其消费者之间的流量访问并提供使用指标的网关** —应用引擎 API 有配额。授权和认证需要有一个 GCP 帐户访问。云端点从 App Engine 应用程序生成 API 和客户端库。

**生命周期管理，管理 API 的设计、开发、部署、版本控制和淘汰流程**—Google development&提供 API。*注意:对于第三方提供的功能，您需要向这些供应商核实*

我不相信我做一个直接的类比是不真诚的！

在我结束谈论 App Engine 的特性之前，我只想简单地谈一下(好吧，一个简单，另一个不简单)两个我觉得特别酷并且恰好符合我的 API 管理观点的特性。URL 获取和云端点。所以，是的，我将在这里偷偷输入字母 E，因为端点实际上被称为 GCP 产品！

[**URL 获取**](https://cloud.google.com/appengine/features/#urlfetch)

这里的特性(服务)的名字非常简洁。这是一种允许您发出 http 或 https 请求并接收响应的服务！这些应用程序可以是其他应用程序引擎应用程序或网络上的其他应用程序。

所以不要轻视它，它是一种让您的应用程序轻松处理 http 和 https 请求和响应的方式。

该服务有一些规则来防止递归请求，即不要自己调用和超时，所以它不会永远等待响应。它还支持同步和异步调用。还有分配的配额。

[**云端点**](https://cloud.google.com/endpoints/)

这个功能超级酷。作为提高开发人员生产力的一种方式，因为它提供了一种从 App Engine 应用程序生成 API 和客户端库的方式。你可能会碰到这被称为 API 后端，这可能会让你挠头，到底是什么意思。我觉得对它更好的描述是作为你的应用程序的 API 服务器，事实上在一些文档中也是这么说的。简而言之，云端点为 Android、iOS 客户端和 JavaScript web 客户端执行业务逻辑和其他功能。它提供了对 App engine 特性的访问(虽然我很确定我不喜欢用特性这个词来描述通过 App Engine 提供的服务)。

因此，如果您仍然不清楚它们是什么，我已经复述了官方文档，对什么是端点以及它们如何删除基本功能的较低级别的编码进行了简明描述，从而使您可以专注于您的 usp:

“云端点创建 RESTful 服务，并使它们可被 iOS、Android 和 Javascript 客户端访问。自动生成客户端库，使连接前端变得容易。内置功能包括拒绝服务保护、OAuth 2.0 支持和客户端密钥管理”

那么，要创建自己的 API 服务器和使用它的应用程序，您到底需要做些什么呢:

1.  首先编写您的 API 代码——这将是外部客户机通过 RPC 可访问的方法的创建
2.  生成客户机库和发现文档甚至有一个[的便捷工具](https://cloud.google.com/appengine/docs/python/endpoints/endpoints_tool)可以帮你做到这一点
3.  [在您的本地开发服务器上测试](https://cloud.google.com/appengine/docs/python/endpoints/test_deploy)
4.  使用 OAuth 限制对方法的访问，然后部署

这只是一个小小的开端，就像我的类比一样，应用引擎特性和端点(这本身就是一个特性)在技术上可以被认为是 API 网关/服务器。