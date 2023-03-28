# Knative v0.18.0 更新

> 原文：<https://medium.com/google-cloud/knative-v0-18-0-update-2045f5d5aba3?source=collection_archive---------4----------------------->

我开始更新我的[技能教程](https://github.com/meteatamel/knative-tutorial)，从技能`v0.16.0`到最新的技能服务`v0.18.0` [版本](https://github.com/knative/serving/releases/tag/v0.18.0)和技能事件`v0.18.1` [版本](https://github.com/knative/eventing/releases/tag/v0.18.1)。

在这篇简短的博文中，我想概述一下我在 Knative `v0.18.0`升级过程中遇到的几个小问题。请注意，我完全跳过了`v0.17`，其中一些变化可能已经在那个版本中发生了。

# Istio 安装

我遇到的最大的变化是 Istio 是如何为 Knative 安装的。在以前的版本中，我只是简单地指向 Knative Serving 的`third_party`文件夹中最新 Istio 版本中的 yaml 文件。

在最新版本中，`third_party`文件夹不再包含 Istio 版本。通过这次[提交](https://github.com/knative/serving/pull/8945/commits/9fb9b37a705254a1449dff362ac5b5c5a8818e79)，我了解到 [net-istio 安装脚本](https://github.com/knative-sandbox/net-istio/tree/master/third_party)是安装 istio 的方式。

考虑到这一点，我更新了我的 Knative 教程中的 install-istio 说明。

# Knative 事件版本 0.18.1

当我第一次安装 Knative Eventing `v0.18.0`时，我遇到了一个奇怪的错误，我无法将 Broker 注入名称空间并创建触发器。原来，Knative 中有一个 [bug](https://github.com/knative/eventing/issues/4165) 导致 eventing webhook 陷入无限崩溃循环。

团队发布了 Knative Eventing `v0.18.1`来解决这个问题，结果我更新了我的设置[配置](https://github.com/meteatamel/knative-tutorial/blob/master/setup/config)来使用 Knative Serving 和 Eventing 的不同版本。因此，重要的是你使用`v0.18.1`开始 Knative Eventing。

就是这样。我其余的设置和教程步骤工作正常。

你可以随时在 Twitter [@meteatamel](https://twitter.com/meteatamel) 上联系我，或者阅读我以前在 [medium/@meteatamel](/@meteatamel) 上的帖子。

*最初发布于*[*https://atamel . dev*](https://atamel.dev/posts/2020/10-12_knative-v0180-update/)*。*