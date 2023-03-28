# 您在企业 Kubernetes 上的成功不仅仅与 Kubernetes 有关

> 原文：<https://medium.com/google-cloud/your-success-with-enterprise-kubernetes-isnt-about-kubernetes-1c2b18dedc9?source=collection_archive---------1----------------------->

像谷歌一样运行不仅仅是 Kubernetes 产品或一个漂亮的用户界面。

![](img/4c092692b47179bdbfcb344748efe211.png)

你不能就这样做。

在我的上一篇文章中，我谈到了我们如何评估技术和我们如何寻找技术来帮助我们的二分法。TL；博士:谷歌长期以来一直将工具和技术创新视为其文化的衍生物，以及尽可能以最佳方式交付的动力。谷歌运营的“魔力”并不在于其令人难以置信的技术，而是渗透到所采用的技术中的谷歌文化，并通过这种采用来加强改进的循环。这种认识是谷歌对其他组织的愿景，也是谷歌向世界发布 Kubernetes 以及通过谷歌 Kubernetes 引擎支持的愿景。

这就是问题所在。如果说 Kubernetes 的成功有什么启示的话，那就是谷歌交付和管理其所有工作负载的方式在各行业和用例中都得到了验证。Kubernetes 不仅仅是一个用于容器编排的开源项目，它是 Google 在任何规模下运行和操作的遗产。能够像谷歌一样运行和操作的想法很诱人，随着 Kubernetes 的采用，这种想法成为了潜在的现实。

这是另一个问题。Kubernetes 的采用率正以令人难以置信的速度上升，似乎不可阻挡，然而采用的痛苦与该平台的纯粹受欢迎程度一样一致。正如我在上一篇文章中关于技术采用的观点是一个公认的痛点，我们正在接受 Kubernetes 很难操作和维护。即使是提供自动化工具使事情变得更容易的 Kubernetes 发行版仍然存在不足。原因如下。

当一个组织选择采用一种技术时，他们在将该技术引入以下受影响的领域时，总是会接受一定程度的开销:

*   **团队技能:**这是一个赋能的问题，也是一个让人们变得更瘦的问题。
*   **组织流程***:** 引进一项技术将为组织提供改进流程的机会。这几乎总是非同小可的，但对于那些选择抓住机会并坚持下去的人来说总是富有成效的。
*   **日常运营的现状:**您要求您的团队(技能有限，见第一条)承担额外的责任，不仅仅是管理和维护新技术，还包括支持该技术所需的一切。无论是基础架构，甚至是为协助日常运营而实施的自动化，它仍然是一种进入您脑海并停留在那里的开销。

******* 我们将重点关注对组织流程的影响，因为这是最重要的，也是最难改变或改进的。谷歌的 Kubernetes 引擎缓解了另外两个问题。方法如下:

*   **团队技能组合&日常运营的现状** : GKE 推出了一个按钮式供应功能，在保证和 SLA 的情况下，立即提供任意数量的企业级 Kubernetes 集群，由谷歌自己的 SRE 团队管理和维护。这为组织的团队提供了一个机会，让他们更专注于使用 Kubernetes 的*，并学习如何使用它，而不是处理任何低级的管道。无需设置，无需采购，也无需自动化管理。结果是，采用企业级 Kubernetes 所能想象到的最小开销。*

GKE 被设计成这样的原因是因为在它存在之前，我们谷歌就已经按照 Kubernetes *的原则和原则运行，并管理类似的系统*。正如我们部署管理和流程编排的最佳实践已经在 Kubernetes 上展示出来供全世界使用一样，我们维护和管理技术本身的最佳实践和方法也已经在 GKE 上展示出来。

[作为一个主要的例子，Borg](https://ai.google/research/pubs/pub43438) 被用作内部开发和部署的服务，团队不必管理甚至自动化他们自己的集群。通过让团队管理他们自己的 Borg，我们没有达到每周 40 亿个集装箱。*即使有最好的自动化，这也不是一种可持续或可扩展的方法。*

> 正如我们在谷歌能够通过我们的采用方法做更多的事情一样，你们在 GKE 也是如此。

这就是像谷歌一样运行的含义。这不是技术本身，而是我们如何以一种更容易大规模采用和利用的方式交付该技术。易于采用为其他两个受影响的领域提供了喘息空间。当我可以配置一个安全的生产级集群并开始在两个命令内使用它时，这对个人、团队和组织来说都是一件令人难以置信的事情。它允许我思考如何将它注入到各种改进过程中，以及如何最好地开始使整个组织在技能集方面能够立即利用这些改进，而无需大幅提升。正如我在上一篇文章中所说的，这里展示了在 Google 上使用企业 Kubernetes 集群有多快:

`$ gcloud container clusters create my-cluster --zone=<pick a zone>`

`$ gcloud container clusters get-credentials my-cluster`

`$ kubectl run something-awesome ....`

> 从 SRE 的角度来看，谷歌将 GKE 的每一个集群都视为生产。

坏消息是，在今天或未来，世界上没有任何技术或工具可以为你改变组织文化。是否愿意走上改进过程的道路完全取决于组织的涉众。但就谷歌的运营方式而言，GKE 可以让这一旅程比其他任何发行版都要容易得多。在我的下一篇文章中，我将向你介绍 GKE 通过灵活的选择来提供更好地管理你的组织的不同方法。我们将查看不同的组织使用案例，并了解 GKE 如何快速、安全、轻松地融入目标愿景。我们也将开始得到更多的手。