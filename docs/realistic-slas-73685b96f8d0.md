# 现实 SLA

> 原文：<https://medium.com/google-cloud/realistic-slas-73685b96f8d0?source=collection_archive---------1----------------------->

## 让我为云开发运维工程师考试做好准备的经历，第二部分

![](img/32bd02d889dfde4891367cb3cce58b35.png)

*披露:我是谷歌的员工。这篇文章反映的观点是个人的，并不代表我的雇主的观点。以下是我的三篇博客的第二部分，* [*为我准备云开发工程师考试*](/@aron_60636/experiences-that-prepared-me-for-the-cloud-devops-engineer-exam-bd7edee4ccaa) *的经历。*

根据我在考试中遇到的经验，设定目标是最具挑战性的话题之一。

挑战涉及设定目标的*人*的动机和想法，而不仅仅是用于实现目标的技术。

> “希望不是策略。”——[传统 SRE 谚语](https://sre.google/sre-book/introduction/)

让我们从 SLA 开始。

几年前，我在咨询一家小型初创公司时遇到了一种情况。他们的网站上有一个“99.99%可用性”的 SLA 我问这个数字的理由是什么，结果发现有人随意选择了它，因为它似乎是行业规范，并且符合他们的云提供商的 SLA。

风险是显而易见的:如果 99.99%纯粹是渴望，他们很有可能无法实现，这会让他们的业务付出代价。

我们决定找一个更现实的数字。

Alex Ewerlö对如何[计算复合 SLA](https://alexewerlof.medium.com/calculating-composite-sla-d855eaf2c655) 有一个很好的概述。不用说，一旦我们更仔细地观察架构、云提供商服务的 SLA，并观察应用程序本身，我们最终得到的数字比 99.99%更低。

这让每个人都松了一口气，因为较低的 SLA 并没有影响初创公司的销售。

他们的客户关心的不是每月退款前的确切分钟数，而是如果停电会发生什么。

这项服务可能会阻止用户登录，所以它需要[失效开放](https://community.microfocus.com/cyberres/b/sws-22/posts/security-fundamentals-part-1-fail-open-vs-fail-closed)以防断电。找到一种优雅的方式来处理失败是一个常见的话题，而 SLA 是一个事后的想法。

尽管任何使用服务的公司都可能遇到 SLA，但重要的是要注意一些全球服务没有或不需要它们:

> 谷歌搜索是一个重要服务的例子，它没有针对公众的 SLA:我们希望每个人尽可能流畅和高效地使用搜索，但我们没有与整个世界签署合同。即便如此，如果搜索不可用，仍然会有后果——不可用会导致我们的声誉受损，以及广告收入下降。许多其他的谷歌服务，比如 Google for Work，确实与他们的用户有明确的 SLA。无论特定的服务是否有 SLA，定义 sli 和 SLO 并使用它们来管理服务都是有价值的。( [SRE 手册，Ch。4 —服务水平目标](https://sre.google/sre-book/service-level-objectives/)

考试中涉及的主题往往集中在 SLO，或它们所基于的 sli 上。他们分享了与上述经历相似的框架，这一经历向我展示了商业利益相关者和工程之间的双向理解。

工程应该关注业务利益相关者，以**定义客户在“服务级别”上需要什么**否则，他们可能会[设定与业务无关的目标](https://bootcamp.uxdesign.cc/operational-focus-why-symptoms-not-causes-e4af0e115e14)，并为此过度投入资源。

在我的例子中，经过反复试验，我们发现稍低的 SLA 不会影响销售。提前预测这种类型的影响是可能的，即使在为没有 SLA 的服务设置内部 SLO 时也是如此。

理解必须是相互的。商业利益相关者应该向工程寻求理解[成本和复杂性的权衡](/@aron_60636/experiences-that-prepared-me-for-the-cloud-devops-engineer-exam-bd7edee4ccaa#ff58)以增加他们的目标，*尤其是*它如何影响发布速度。

在我的例子中，假设企业坚持坚持其最初的 99.99%的索赔。这会增加多少成本，又会阻碍多少发展？

事实上，风险是存在的，更适度的索赔是更安全的赌注，这最终是合理的商业推理。这需要一些工程师*验证*逻辑并识别隐藏的紧张。

确定一个技术决策如何影响客户、您的业务以及构建和交付产品的团队，这将比狭隘地关注系统内部行为更有助于这些主题。

你们有类似的故事可以分享吗？看看[可靠性工程](https://sites.google.com/view/reliability-discuss/)，它有一个精益咖啡的形式，你可以提出讨论的话题，人们可以投票选出他们最喜欢的。那个小组中关于 SLO 的讨论给了我一个很好的心理模型，它在考试中帮助了我，并帮助我提出了我的帖子[为什么要优先考虑症状而不是原因](https://bootcamp.uxdesign.cc/operational-focus-why-symptoms-not-causes-e4af0e115e14)。

在我下周的最后一篇文章中，我们将看看在不可避免的失败情况下该怎么做。

[![](img/cb2ba37628dbf4bdb5ecc3320cc38d47.png)](https://www.credential.net/0dcec0e8-4eaf-466f-88ea-9966aa51b7cf#gs.irxraz)