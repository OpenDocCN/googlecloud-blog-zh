# 对话流体验

> 原文：<https://medium.com/google-cloud/the-dialogflow-experience-1a19e64187bd?source=collection_archive---------1----------------------->

## 我作为 bot 开发者参与 Dialogflow CX 竞赛的经历

![](img/0f3a61bf952ed28f6e8aa5143f3d2c93.png)

在 [Unsplash](https://unsplash.com?utm_source=medium&utm_medium=referral) 上拍摄的 [ThisisEngineering RAEng](https://unsplash.com/@thisisengineering?utm_source=medium&utm_medium=referral)

我作为一名 bot 开发者参加了 Dialogflow CX 竞赛。我想要的是 slack 社区能够将 dialogflow cx 与他们的 slack 机器人一起使用。以下是我使用 Dialogflow CX 构建一个机器人的经验和教训，以及整个事件的主要收获！

# 什么是对话流 CX？

谷歌的[对话流 CX](https://cloud.google.com/dialogflow/cx/docs/basics) 是一个对话式人工智能开发平台，适合创建聊天机器人流。这个平台的独特性和简单性在于通过具有意图和实现的页面来定义与用户的对话流。

> 比方说，你想要一个安排出租车的机器人。你可以创建/使用一个代理，它可以为你与人类用户对话。
> 
> 现在，代理将拥有页面或流程，它们主要是代理可以覆盖的主题，相互链接，每个页面或流程都有路线/意图、实体、参数和完成。

# 为什么我们需要一个用于 slack 机器人的对话式人工智能？

在 slack 上创建一个 [bot](https://api.slack.com/apps?new_app=1) 非常容易，它还为任何定义的特定 webhook 提供事件订阅，这些事件可以在 Lambda 函数上运行，该函数可以对发送的事件做出响应，并将适当的消息发布回 Slack。

> 现在，lambda 函数可以进一步利用 Dialogflow CX 来处理消息事件并获取对话的意图。如果没有 NLP，你可能会在 lambda 中定制代码，检查主要关键字，触发响应 webhook。

对话流 CX 为你抽象出自然语言处理。

> 以上述安排 cab 的例子为例，您的 Dialogflow CX 代理是管理您的最终用户对话的虚拟代理。该代理通过自然语言处理将用户文本或音频转换为结构化数据，您的应用程序和服务可以在整个对话过程中理解这些数据。

# 我们如何知道某个事件何时发生在 slack 上？

Slack 有两种向定义的应用程序发送订阅事件的模式。

一个是事件 API，一个是 socket 模式。

> 在 event API 中，您需要定义一个 webhook url，slack 将在这个 url 上发送事件。
> 
> 你可以订阅机器人提及，这意味着每当用户使用@botname 提及机器人时，它就会将消息事件发送到你的应用程序。你会希望你的应用和用户在同一个线程中回复，slack 确实为你提供了 ts 来唯一地识别用户的线程。
> 
> 您可能想要在应用程序的主目录中与其聊天，需要订阅聊天消息事件。

# 对话看起来自然吗？

用户和代理人之间的对话是这样的:

> 山姆:嗨，我想叫辆出租车！
> 
> 代理人:嗨，山姆！当然，你想什么时候取？
> 
> 山姆:15 分钟后
> 
> 代理人:当然，取货的地方在哪里？
> 
> 山姆:是加里哈特
> 
> 经办人:在哪下车？
> 
> 山姆:在希亚姆巴扎尔
> 
> 经办人:你想要高级车还是经济型车？
> 
> 山姆:经济舱
> 
> 经办人:这要花你 350 卢比

使用 dialogflow 会话上下文，您可以维护 bot 和用户之间的这种会话流。

# 所以最终这一切都归结为…什么？

使用 Dialogflow CX 你可以得到谈话的意图。使用 dialogflow CX 会话上下文，您可以维护 bot 和用户之间的会话流。使用这个 SDK，您可以在 slack 上拥有所有这些。

# 这个黑客，有助于将对话式人工智能——dialog flow CX 带到 Slack！

## 听起来很酷，但是你为什么需要捐款呢？

*   如上所述，Dialogflow CX 和 slack 都提供了大量的特性，这些特性可以在 SDK 中充分利用，从而为 Slack 社区建立一个强大的基础。
*   此外，目前，SDK 支持应用引擎或云功能部署，可以扩展到使用其他云提供商！
*   如果你觉得有帮助，也欢迎你添加任何问题/扩展。

## 太好了，太好了，我该怎么做贡献呢？我的贡献会被接受吗？

你可以先从检查 [SDK](https://github.com/Sampriti-Mitra/dialogflow-slack-sdk) 开始。也请仔细阅读[投稿](https://github.com/Sampriti-Mitra/dialogflow-slack-sdk/blob/main/CONTRIBUTING.md)指南。
所有代码和非代码贡献都是受欢迎的，只要它们遵循贡献指南，都将被接受。

你还在等什么？立即投稿并赢取您的 [Hacktoberfest](https://hacktoberfest.digitalocean.com/) 奖品！