# 关于谷歌云发布/订阅，我希望知道的事情:第 1 部分

> 原文：<https://medium.com/google-cloud/things-i-wish-i-knew-about-google-cloud-pub-sub-852fac1ffbc6?source=collection_archive---------0----------------------->

发布/订阅似乎是一个简单的话题，但实际上有相当多的微妙之处需要考虑。可能会忽略一些细节，而这些细节可能会让你的体验更流畅、更便宜、更少挫败感。为此，Google Cloud Pub/Sub 客户端库的开发人员关系团队正在启动一个博客系列，提供一些您可能会觉得有用的信息和技巧。

事不宜迟…！

![](img/11d37575fb55650d219ab8bd12487ffd.png)

# 什么是 Google Cloud Pub/Sub？

## Google Cloud Pub/Sub 是一个托管的、全球可用的消息服务，可以根据需求自动扩展。

为了从最基本的角度理解该服务，请熟悉四个关键组件:主题、订阅、发布者客户端和订阅者客户端。

*   发布/订阅客户端向**主题**发布消息。
*   发布/订阅客户端接收来自**订阅**的消息。您可以为同一主题创建多个订阅，在这种情况下，每个订阅都会获得该主题中所有消息的副本。一些初学者认为人们会从**主题**中提取信息，这是不可能的——**主题**仅用于发布。**订阅**是您接收所有消息的地方。
*   您的应用程序可以创建多个**发布/订阅客户端**来跨不同的进程发布到一个主题。(下面提到了一些注意事项。)
*   您的应用程序还可以为单个订阅创建多个**订阅者客户端**，在这种情况下，消息传递会在它们之间实现负载平衡。消息被发送到**订阅**的一个客户端，而不是所有客户端。

# 如何调用发布/订阅 API？

## 你可以通过[谷歌云控制台](http://console.cloud.google.com)和[云 SDK](https://cloud.google.com/sdk) 以及[官方客户端库](https://cloud.google.com/pubsub/docs/quickstart-client-libraries)使用 Pub/Sub。访问 Pub/Sub API 最常见的方式是通过客户端库，有 8 种语言可用( [C++](https://cloud.google.com/pubsub/docs/quickstart-client-libraries#cpp) 、 [C#](https://cloud.google.com/pubsub/docs/quickstart-client-libraries#c) 、 [Go](https://cloud.google.com/pubsub/docs/quickstart-client-libraries#pubsub-client-libraries-go) 、 [Java](https://cloud.google.com/pubsub/docs/quickstart-client-libraries#java) 、 [Node](https://cloud.google.com/pubsub/docs/quickstart-client-libraries#node.js) 、 [PHP](https://cloud.google.com/pubsub/docs/quickstart-client-libraries#php) 、 [Python](https://cloud.google.com/pubsub/docs/quickstart-client-libraries#pubsub-client-libraries-python) 、 [Ruby](https://cloud.google.com/pubsub/docs/quickstart-client-libraries#ruby) )。

这些库提供了符合语言习惯的方法来创建主题、发布主题消息、创建主题订阅以及从订阅接收消息。这些客户端是手写的，构建在[自动生成的库](https://github.com/googleapis/java-pubsub/tree/master/proto-google-cloud-pubsub-v1/src/main)之上，这些库公开了可从发布/订阅服务器获得的基本 RPC 方法。这使我们能够添加一些有用的功能，主要是提高吞吐量、最小化延迟，并使处理传入消息变得更容易。

因为库缓存像认证令牌这样的信息，所以我们建议在一个进程中对每个资源只实例化一个 Pub/Sub 客户机实例，而不是创建多个客户机。这因语言而异；例如，Java 使用单独的客户端类，而 Node 有一个 PubSub 类，应该为每个进程、每个 Pub/Sub 服务器连接创建；然后用于创建发布和订阅的客户端。

在 Node 中，它可能看起来像这样:

```
let pubSubClient, topic;async function main() {
  pubSubClient = new PubSub();
  topic = pubSubClient.topic(‘myTopic’);
  topic.publish(Buffer.from(‘test!’));
  await doOtherWork();
}async function doOtherWork() {
  // ...
  topic.publish(Buffer.from(‘also test!’));
  // ...
}main().catch(console.error);
```

每个库都有一个功能更全的 Quickstart，用于更全面的示例；比如[节点快速启动](https://github.com/googleapis/nodejs-pubsub#quickstart)。

如果您想更好地控制我们的客户端库中没有的特性，您可以使用自动生成的库、 [REST](https://cloud.google.com/pubsub/docs/reference/rest) 或 [RPC](https://cloud.google.com/pubsub/docs/reference/rpc) 存根来构建自己的客户端库。但是，我们强烈建议您寻找一种客户端库本身支持的方法。如果找不到，请在 GitHub 或 [Buganizer](https://issuetracker.google.com/issues?q=componentid:187173) 上的相应库添加一个错误报告。

# 还会有更多

感谢您的阅读，请期待来自发布/订阅开发者关系团队的未来帖子(Part [2](/google-cloud/things-i-wish-i-knew-about-google-cloud-pub-sub-part-2-b037f1f08318) ， [3](/google-cloud/things-i-wish-i-knew-about-pub-sub-part-3-b8947b49224b) )，这些帖子将帮助您更加轻松地使用发布/订阅！我们也很乐意听到你关于你想讨论的话题，或者你对这个帖子有什么意见！