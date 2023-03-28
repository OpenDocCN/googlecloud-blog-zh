# Knative v0.12.0 更新

> 原文：<https://medium.com/google-cloud/knative-v0-12-0-update-36f9ee1d49fb?source=collection_archive---------4----------------------->

每 6 周发布一次的 Knative 版本很难跟上。我终于把我的 [Knative 教程](https://github.com/meteatamel/knative-tutorial)更新到了最新的 Knative v0.12.0。在这篇博文中，我想概述一下我观察到的一些差异。

## 美味的服务

Knative Serving 在最近的版本中已经相当稳定了，v0.12.0 也不例外。我不需要专门为这个版本更新我的教程。

## 破坏性事件

[Knative Eventing v0.12.0](https://github.com/knative/eventing/releases/tag/v0.12.0) 更改了 Knative Eventing 包的默认 yaml。现在，它们在`eventing.yaml`(以前是`release.yaml`)下面，这是你需要指向安装事件的 yaml。这是有道理的，因为它更符合 Knative 发球和它的`serving.yaml`。

## GCP 的胜利

v0.12.0 的一个主要区别是 Google Cloud 发布/订阅消息是如何被拉入 Knative 的。你可能还记得，GCP 有一个专门针对谷歌云事件的 [Knative 项目。这个项目也有了一个新的版本，在 GCP 的版本 0.12.0 中，你安装和设置 GCP 的版本的方式有了很大的改变。您需要:](https://github.com/google/knative-gcp)

*   按照[安装带 GCP 的 Knative](https://github.com/google/knative-gcp/blob/master/docs/install/README.md)页安装组件。
*   按照[安装启用发布/订阅的服务帐户](https://github.com/google/knative-gcp/blob/master/docs/install/pubsub-service-account.md)页面配置启用发布/订阅的服务帐户。
*   按照 [CloudPubSubSource 示例](https://github.com/google/knative-gcp/blob/master/docs/examples/cloudpubsubsource/README.md)设置读取发布/订阅消息的源。

我应该提到，CloudPubSubSource 是从早期版本中重新命名的 PubSub。 [CloudPubSubSource 示例](https://github.com/google/knative-gcp/blob/master/docs/examples/cloudpubsubsource/README.md)将发布/订阅消息直接路由到服务。我不喜欢这种方式。在我的 Knative 教程中，我稍微修改了一下，让它通过 Broker(下面会详细介绍)。

## 实用教程

在 Knative 教程中，我已经对 v0.12.0 进行了更新。

在[设置](https://github.com/meteatamel/knative-tutorial/tree/master/setup)页面，我提供了有用的指令和脚本来设置 GCP 的 Knative 服务、事件和 Knative。希望您可以运行它们来正确安装所有东西。

[在发布/订阅触发的服务示例](https://github.com/meteatamel/knative-tutorial/blob/master/docs/pubsubeventing.md)中，我展示了如何将发布/订阅消息读入默认名称空间中的代理，然后通过触发器将它们路由到服务。看看这个。