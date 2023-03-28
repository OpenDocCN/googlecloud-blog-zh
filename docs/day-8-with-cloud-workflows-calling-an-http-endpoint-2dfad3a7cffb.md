# 云工作流的第 8 天:调用 HTTP 端点

> 原文：<https://medium.com/google-cloud/day-8-with-cloud-workflows-calling-an-http-endpoint-2dfad3a7cffb?source=collection_archive---------0----------------------->

是时候做一些非常方便的事情了:从 Google Cloud Workflows 定义中调用 HTTP 端点。无论是调用特定于 GCP 的 API(如 ML APIs)、其他产品(如 Cloud Firestore)的 REST APIs，还是调用您自己的服务、第三方、外部 API，此功能都可以让您将业务流程与外部世界相连接！

在深入下面的细节之前，让我们在下面的视频中看看如何调用 HTTP 端点:

默认情况下，当创建一个新的工作流定义时，会提供一个默认的片段/示例供您参考。在这篇文章中，我们将研究一下它。实际上有两个 HTTP 端点调用，后者依赖于前者:第一步(getCurrentTime)是一个返回星期几的云函数，而第二步(readWikipedia)在维基百科中搜索关于星期几的文章。

```
- getCurrentTime:
    call: http.get
    args:
        url: https://us-central1-workflowsample.cloudfunctions.net/datetime
    result: CurrentDateTime
```

getCurrentTime 步骤包含 http.get 类型的 call 属性，用于向 API 端点发出 HTTP GET 请求。您可以调用:http.get 或 call: http.post。对于其他方法，您必须调用:http.request，并在 args 下添加另一个键/值对，使用方法:GET、POST、PATCH 或 DELETE。现在，在 args 下，我们只放 HTTP 端点的 URL。最后一个键将是结果，它给出了一个新变量的名称，该变量将包含我们的 HTTP 请求的响应。

让我们用我们的星期几搜索查询打电话给维基百科:

```
- readWikipedia:
    call: http.get
    args:
        url: [https://en.wikipedia.org/w/api.php](https://en.wikipedia.org/w/api.php)
        query:
            action: opensearch
            search: ${CurrentDateTime.body.dayOfTheWeek}
    result: WikiResult
```

call 和 args.url 也是如此，但是，我们有一个查询，您可以在其中为 Wikipedia API 定义查询参数。还要注意我们如何传递来自上一步函数调用的数据:current date time . body . day oftheweek。我们检索前一个调用的响应体，并从那里获得结果 JSON 文档中的 dayOfTheWeek 键。然后我们返回 WikiResult，这是新 API 端点调用的响应。

```
- returnOutput:
    return: ${WikiResult.body[1]}
```

然后，最后一步是返回我们搜索的结果。我们检索响应的主体。响应的主体是一个数组，第一项是搜索查询，第二项是下面的文档名称数组，这是我们的工作流执行将返回的内容:

```
[
  "Monday",
  "Monday Night Football",
  "Monday Night Wars",
  "Monday Night Countdown",
  "Monday Morning (newsletter)",
  "Monday Night Golf",
  "Monday Mornings",
  "Monday (The X-Files)",
  "Monday's Child",
  "Monday.com"
]
```

因此，我们的整个工作流能够一个接一个地编排两个独立的 API 端点。云工作流不是有两个 API 通过某种消息传递机制耦合在一起，或者更糟，通过显式调用一个或另一个，而是在这里组织这两个调用。这是一种编排方法，而不是服务的编排(参见我之前关于[编排与编排](https://glaforge.appspot.com/article/orchestrating-microservices-with-cloud-workflows)的文章，以及我同事关于[利用云工作流实现更好的服务编排](https://cloud.google.com/blog/topics/developers-practitioners/better-service-orchestration-workflows)的文章)。

回到 API 端点调用的细节，下面是它们的结构:

```
- STEP_NAME:
    call: {http.get|http.post|http.request}
    args:
        url: URL_VALUE
        [method: REQUEST_METHOD]
        [headers:
            KEY:VALUE ...]
        [body:
            KEY:VALUE ...]
        [query:
            KEY:VALUE ...]
        [auth:
            type:{OIDC|OAuth2}]
        [timeout: VALUE_IN_SECONDS]
    [result: RESPONSE_VALUE]
```

除了 URL、方法和查询之外，请注意，您还可以传递头和主体。还有一个与 GCP API 一起工作的内置认证机制:认证是透明完成的。如果您希望快速失败，而不是永远等待永远不会到来的响应，您也可以指定以秒为单位的超时。但是，在我们即将发表的一些文章中，我们将回头讨论错误处理。

【http://glaforge.appspot.com】最初发表于[](http://glaforge.appspot.com/article/day-8-with-cloud-workflows-calling-an-http-endpoint)**。**