# 1 小时迁移#1: SQS 到 GCP 的云发布/订阅

> 原文：<https://medium.com/google-cloud/1-hour-migrations-1-sqs-to-gcps-cloud-pub-sub-105bcac63318?source=collection_archive---------0----------------------->

消息队列在当今可伸缩的分布式基础设施中扮演着重要的角色。亚马逊网络服务(AWS)提供 SQS(简单队列服务)，谷歌云平台(GCP)提供云发布/订阅。

这些都是用于发布和消费消息的可管理的、可伸缩的、可靠的服务。在这篇文章中，我想从高层次上讨论如何将你的应用从使用亚马逊的 SQS 迁移到谷歌的发布/订阅。

## 伙计，我的队伍呢？

在我们深入探讨之前，让我们将一些术语从 SQS 映射到 Pub/Sub。

在 SQS，您处理队列，而在发布/订阅中，您处理主题和订阅。你可能会问，等等，为什么我的队列变成了两个独立的实体？这与 Pub/Sub 的工作方式有关。Pub/Sub 中实际的类似队列的范例由一个主题表示，但是如果没有消费者为该主题创建和订阅一个(或多个)订阅，您将无法做太多事情。

让我们看看下面的 SQS 队列:

1.  **名称:我的队列**
2.  **地区:美国东部-1**
3.  **消息保留时间:12 天**
4.  **先进先出:否**
5.  **重驱动策略:否(无死信队列)**
6.  **最大有效载荷大小:256KB**
7.  **消息传递延迟:无**

向云发布/订阅迁移的第一步是:

1.  创建一个名为“myqueue”的主题
2.  为该主题创建一个名为“mysubscription”的订阅。

请注意:

*   不需要定义区域，因为发布/订阅主题是全局的。
*   发布/订阅中的消息保留期(在本文发布之日)是 7 天，所以我们不能像 SQS 上的配置那样达到 12 天。

## 发布消息(即让他们拥有它！)

在 SQS，发布消息就像调用 SendMessage API 一样简单，只需要队列 URL 和消息有效负载。在 Cloud Pub/Sub 中，您可以只使用主题名称和有效负载来调用“publish”方法。很简单，但是请注意，cloudpub/Sub API 将返回一个 future，您需要等待它来解决，直到您可以保存发送的消息 Id。

## 接收消息(即收到消息！)

要从 SQS 轮询消息，您可以以一种简单的方式使用 AWS api:

```
response **=** client**.**receive_message(
    QueueUrl**="https://somearn/myqueue"**,
    MaxNumberOfMessages**=**1,
    VisibilityTimeout**=**10,
)
```

现在，在云发布/订阅中，您可以使用消息轮询或消息推送(在[https://cloud.google.com/pubsub/docs/subscriber](https://cloud.google.com/pubsub/docs/subscriber)了解更多信息)。

为了简单起见，我们将使用轮询机制，但是请注意我使用的是异步方式:

```
subscriber = pubsub_v1.SubscriberClient()
subscription_path = subscriber.subscription_path(
    'myproject', 'mysubscription')

def message_callback(message):
    # Do something with the message here
    message.ack()

subscriber.subscribe(subscription_path, callback=message_callback)
```

因为这里的轮询是非阻塞的，如果这是进程唯一要做的事情，你将需要一个主循环来保持它的活动。

# 结论

在这篇文章中，我介绍了一个从简单的 SQS 到云发布/订阅的迁移场景。当然，还有更复杂的用例、架构和最佳实践，我会在接下来的文章中介绍。

更多阅读:

云发布/订阅:[https://cloud.google.com/pubsub/docs/](https://cloud.google.com/pubsub/docs/)

SQS 和云 Pub/Sub:[https://Cloud . Google . com/docs/compare/AWS/application-services](https://cloud.google.com/docs/compare/aws/application-services)