# 无服务器事件:SinkBinding 101

> 原文：<https://medium.com/google-cloud/serverless-eventing-sinkbinding-101-12d9ed31a76f?source=collection_archive---------3----------------------->

![](img/acfbdcf5ee2e6c66bb0ac4cf063bf9dd.png)

无服务器事件

您正在编写一个应用程序，试图以近乎实时的方式分析电视直播中的推文。在这种情况下，5 小时前发的推文对你没有任何意义。你会怎么做？

我们需要创建一个应用程序，能够收集事件，并发送它们以供摄取或消费。这是生产者-消费者模式。一个进程创建一个事件并传输它，而另一个进程接收事件并对数据进行处理。

[Knative Eventing](https://knative.dev/docs/eventing/) 允许通过 [SinkBinding](https://knative.dev/docs/eventing/samples/sinkbinding/) 创建第一方事件源。用最简单的术语来说，SinkBinding 负责将生产者与消费者匹配起来。在这种情况下，您的消费者就是“事件接收器”。

消费者可以是任何使用 PodSpec 的 Kubernetes 资源。这可以是代理集、作业、有状态集，甚至是一个无效服务的部署。如果您不是 Knative 的新手，那么您可能更了解 ContainerSource。ContainerSource 正在回归 Knative Eventing，但更好的解决方案是 SinkBinding。

ContainerSource yaml 包含接收器绑定和部署定义。虽然这更简单，但它也限制了 Knative Eventing 可以用作源的内容。通过将部署定义从接收器定义中分离出来，我们打开了可能性。

我在 GitHub 有一个教程[。在这里，您将创建一个小的 Python 应用程序，它将每 30 秒提取 25 条 tweets，然后将它们发送到一个 Knative 服务，该服务将简单地记录所有传入的消息。](https://github.com/TheJaySmith/serverless-eventing/tree/master/tutorials/twitter-sink-binding)

您将需要在 Twitter 中创建一个[应用程序，以获得完成本教程所需的 API 密钥。GitHub 资源库的 README 文件中详细介绍了所有内容。试一试，然后回来我们谈谈。](https://developer.twitter.com/en/apps)

好吧，你试过了？不酷吗？现在你可能会问自己为什么这很重要。毕竟，难道不能编写这些应用程序来利用某种消息总线，比如 RabbitMQ 或 Kafka 吗？

当然，但是我们在这里谈论的是无服务器。我们正在讨论寻找简化开发人员体验的方法，并要求开发人员或操作人员维护额外的工具来使应用程序工作，这使我们远离了无服务器的目标。

肯定有云提供商提供这些工具的完全托管版本，但完全托管！=无服务器。您已经将维护需求排除在外，但是现在我们必须绑定源和接收器。

Knative Eventing 允许您以声明方式将源绑定到它们的接收器。如果你看代码，你不需要硬编码你想发送代码的地方或者你从哪里接收代码。您也不必导入特殊的库来连接到您的源或接收器。

事件源的代码使用了`K_SINK`环境变量。Knative 使用这个变量来知道将流量路由到哪里。对于我的消费者，我创建了一个接收所有传入流量的端点。

在告诉 Knative 如何安排我的活动时，SinkBinding YAML 就是我所需要的一切。这展示了如何简化将事件源绑定到它们的接收者。

这是一个非常简单的例子，事件直接从源发送到接收器。在未来的教程中，我们将向您展示当您想要为您的事件创建一个更健壮的路由时，经纪人和渠道的力量。

*原载于 2020 年 4 月 20 日【https://thejaysmith.com】[](https://thejaysmith.com/titles/blogroll/serverless-eventing-sinkbinding-101/)**。***