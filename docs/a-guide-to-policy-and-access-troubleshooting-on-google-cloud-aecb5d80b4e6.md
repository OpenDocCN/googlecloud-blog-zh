# 谷歌云政策和访问故障排除指南

> 原文：<https://medium.com/google-cloud/a-guide-to-policy-and-access-troubleshooting-on-google-cloud-aecb5d80b4e6?source=collection_archive---------0----------------------->

## 只要知道您有什么故障诊断工具以及何时使用它们，就成功了一半。

在我们的生活中，每天我们都会遇到问题。一些大的，一些小的，一些平凡的，偶尔，一些与谷歌云有关。问题是不可避免的，而且总是如此，但诀窍是理解、解决和学习，这样你就可以防止再次遇到那个特定的问题及其变体。即使你不能防止它再次发生，那么下次你应该希望知道如何快速识别问题，适应和解决。

要有效地解决任何问题，您需要具备快速识别和解决问题的工具和技能。让我们来看一个我们可能都在某个时候遇到过的典型问题

把你的手机放错了地方。

这是一个经常发生在很多人身上的问题，有很多方法可以找到你的手机，从回忆你最后一次看到你的手机的地方开始。该流程图显示了尝试定位您的电话的典型流程。在这种情况下，拥有一个工具“找到你的手机”应用程序，并知道何时使用它来帮助你快速找到你的手机，有助于减少如果你没有立即找到它，接下来该做什么的辛劳。

![](img/bb10901c5bc35e615e438f5254d84df1.png)

这与解决任何问题没有什么不同。在升级到客户服务之前，谷歌云有许多工具可以帮助你自助，关键是知道什么时候使用什么。我们从如何帮助客户解决与访问相关的问题开始。

在[故障排除政策和 Google Cloud](https://cloud.google.com/architecture/troubleshooting-policy-and-access-problems) 上的访问问题中，我们首先关注您可能会遇到的类以及您可以用来帮助自己诊断它们的工具。

对于类，我们指的是实施约束的一组系统控件。例如，一个类别可能是实施访问控制的 IAM 策略，而另一个类别可能是拒绝网络连接的低级防火墙规则。这些是您可能会遇到的控制系统，但是您需要排除故障的工具非常不同，并且是特定于类别的。仅仅知道类是不够的，然而，有一种方法和层次来识别首先要研究的类。例如，如果您连接到 Spanner 的内部应用程序抛出速率限制异常，您将根据错误立即知道首先调查哪个类(在这种情况下，不是 VPC 防火墙或 IAM，而是基于配额的约束)。

云支持采用这些相同的方法来帮助客户解决访问问题。我们评估类别，然后选择适当的工具来帮助诊断和解决问题。有时我们会为该类提供工具(例如 IAM 策略故障排除程序)，而在其他时候，最好的工具是现成的。这是把问题看成一个难题，并选择正确的工具集来帮助把所有的碎片放在一起。故障诊断之路可以带你深入(“[丢失 DNS 数据包的案例:一个谷歌云支持故事](https://cloud.google.com/blog/topics/inside-google-cloud/google-cloud-support-engineer-solves-a-tough-dns-case)”)，而在其他时候，该工具很容易描述这个问题。不管怎样，把这看作一个难题有助于建立心态

了解使用哪个类和工具有时是通过平台识别模式的经验获得的。为了帮助解决这一问题，我们还发布了[用于解决 Google Cloud](https://cloud.google.com/architecture/troubleshooting-policy-and-access-problems-use-cases) 上的访问问题的用例，通过一些用例引导您了解管理 Google Cloud 组织时可能遇到的类别和常见模式

如上所述，问题会一直存在。这是一个管理和减轻它们的问题，这是诀窍。为此，我们希望继续增加故障排除指南和工具，以帮助我们的客户将更多的时间放在开发更好的应用程序和系统上，而不是故障排除上！如果您有任何想法，请通过您的客户团队或通过[公共问题追踪器](https://developers.google.com/issue-tracker)告诉我们