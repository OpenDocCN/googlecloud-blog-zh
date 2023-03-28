# 在 Google Kubernetes 引擎上持续交付 HashiCorp Vault:简介

> 原文：<https://medium.com/google-cloud/continuous-delivery-of-hashicorp-vault-on-google-kubernetes-engine-introduction-fc1c106ef50c?source=collection_archive---------2----------------------->

**这是一个系列的第一部分:** [**索引**](/@blzysh/continuous-delivery-of-hashicorp-vault-on-google-kubernetes-engine-bcbf4e75f0f6)

# 概述:

你知道那句谚语吗？是的，他们知道。让一个系统运行起来通常很容易。有时候太容易了，尤其是在云中。也就是说，如果你没有仔细考虑如何在云中交付东西，你最终会得到一堆复杂且不可支持的环境，这些环境是一年前离开公司的“Bob”构建的。几年后，您的生产问题将会随着您的服务的扩展而快速增长。

我在这里的目的是谈论一种支持生产系统的方法，使用我过去一年在谷歌云平台上工作时学到的一些东西，[连续交付](https://continuousdelivery.com)以及向比我聪明得多的人学习。

我很幸运成为一个产品团队的一员，这个团队相信[持续交付](https://continuousdelivery.com)、持续改进以及开发人员应该进行操作和测试的理念。虽然我专注于操作，但我认为自己是开发团队中的一名开发人员。目前，我是唯一一个主要关注谷歌云平台运营方面的开发人员。**我试图尽我所能将架构、IaC、自动化、测试、安全、备份、恢复&可观察性整合到运营交付渠道中。**

在这一系列的帖子中，你会看到一些并不太好的东西，它们甚至可能是完全错误的。他们那样做可能有原因，也可能没有。没关系，我会给你这句话:

> [连续发货](https://continuousdelivery.com)不是魔术。它是关于持续的、每日的改进——通过遵循启发性的“如果疼，就更频繁地做，并把疼往前推”来追求更高的性能的持续训练

毫不奇怪，我喜欢这句话，因为我是一个疯狂的混蛋 [CrossFitter](https://www.crossfit.com) 和 [GRT](https://www.goruck.com/tough) 。

# 哈希公司金库:

遵循[连续交付](https://continuousdelivery.com)实践的定制开发的应用程序需要产生一个不可变的工件，它可以被部署到各种环境中。这显然是做不到的，如果我们将配置和秘密烘焙到工件中也不是一个好主意。相反，这些东西需要在运行时提供给工件。这就是 Vault 发挥作用的地方，也是为什么我要在谷歌云平台上的谷歌 Kubernetes 引擎上构建一个 Vault 基础设施。

[*Google Kubernetes 引擎*](https://cloud.google.com/kubernetes-engine) *是一个托管的、生产就绪的环境，用于部署容器化的应用程序。*

[*hashi corp Vault*](https://www.vaultproject.io)*保护、存储并严格控制对令牌、密码、证书、API 密钥和现代计算中其他秘密的访问。*

我已经为 Seth Vargo 的 GitHub repo 创建了一个 fork，它实际上是 Kelsey Hightower 的 GitHub repo 的 fork。

我的 fork 中的一些主要差异是，我将使用[外部 dns](https://github.com/kubernetes-incubator/external-dns) 来同步 Kubernetes ingress 资源与 [Google Cloud DNS](https://cloud.google.com/dns) 以及 [cert-manager](https://github.com/jetstack/cert-manager) 来自动管理和发布来自 [Let's Encrypt](https://letsencrypt.org) 的 TLS 证书。这里显然有一些权衡，您必须决定它们是否有意义，是否对您的用例足够安全。

一些更小的东西，比如:

*   [谷歌云存储](https://cloud.google.com/storage)用于[远程状态](https://www.terraform.io/docs/state/remote.html)配置[地形](https://www.terraform.io)。
*   [地区谷歌 Kubernetes 引擎集群](https://cloud.google.com/kubernetes-engine/docs/concepts/regional-clusters)
*   [全局](https://cloud.google.com/kms/docs/locations) [谷歌云密钥管理服务](https://cloud.google.com/kms)
*   [Google 容器注册表](https://cloud.google.com/container-registry)用于图像的“本地”存储和[容器分析](https://cloud.google.com/container-registry/docs/get-image-vulnerabilities) ( [W](https://github.com/lzysh/ops-gke-vault/issues/1) IP)

虽然这篇文章关注的是 Vault，但是这里的思想和代码对于这种类型的架构是可重用的。

[**第二部分- >**](/@blzysh/continuous-delivery-of-hashicorp-vault-on-google-kubernetes-engine-google-architecture-4fb88900d23d)