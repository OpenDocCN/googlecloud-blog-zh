# 将 Monolith 迁移到 Google Kubernetes 引擎——概述

> 原文：<https://medium.com/google-cloud/migrating-a-monolith-to-google-kubernetes-engine-an-overview-785f2cbe5c62?source=collection_archive---------1----------------------->

## 在云中烹饪

作者:[普里扬卡·韦尔加迪亚](https://twitter.com/pvergadia)，[卡特·摩根](https://twitter.com/carterthecomic)

![](img/c4bcbde49cdae22ad3e3f5c644ded927.png)

# 介绍

*[***获取云端烹饪***](/@pvergadia/get-cooking-in-cloud-an-introduction-5b3b90de534e)*是一个[博客](/@pvergadia/get-cooking-in-cloud-an-introduction-5b3b90de534e)和[视频](https://www.youtube.com/playlist?list=PLIivdWyY5sqIOyeovvRapCjXCZykZMLAe)系列，帮助企业和开发者在 Google Cloud 上构建商业解决方案。在这第三个迷你系列中，我们将讲述 ***将一个整体迁移到谷歌 Kubernetes 引擎(GKE)*** 。将整体迁移到微服务可能会令人生畏。一旦你决定接受它，你需要考虑什么？继续读...**

**在这些文章中，我们将带您了解将 monolith 迁移到微服务的整个过程、迁移流程、首先迁移什么、迁移的不同阶段以及如何处理数据迁移。我们这些文章的灵感来自于[这篇解决方案文章](https://cloud.google.com/solutions/migrating-a-monolithic-app-to-microservices-gke)。我们将以一个真实的客户故事来结束这一切，在一个真实的应用程序中完成这些步骤。**

**以下是这部迷你剧的所有文章，供你查阅。**

1.  **将一块巨石迁移到 GKE:概述(本文)**
2.  **[将一块巨石迁移到 GKE:迁移过程](/google-cloud/migrating-a-monolith-to-google-kubernetes-engine-gke-migration-process-2de2f51986a2)**
3.  **[将整块材料迁移到 GKE:分阶段迁移](/google-cloud/migrating-a-monolith-to-google-kubernetes-engine-gke-migrate-in-stages-7286ec26689c)**
4.  **将一块巨石迁移到 GKE:先迁移什么？**
5.  **将整体迁移到 GKE:数据迁移**
6.  **[将一个整体迁移到 GKE:客户案例](/google-cloud/migrating-a-monolith-to-google-kubernetes-engine-gke-customer-story-c35c320325eb)**

**在第一篇文章中，我们将向您介绍单片和微服务的概念..所以，继续读下去吧！**

# **你会学到什么**

*   **什么是独石**
*   **什么是微服务**
*   **为什么要转向基于微服务的架构**
*   **三种主要的迁移模式**

# **先决条件**

*   **谷歌云的基本概念和结构，这样你就可以识别产品的名称。**
*   **查看云系列中 [Get Cooking 的介绍。](/@pvergadia/get-cooking-in-cloud-an-introduction-5b3b90de534e)**

# **看看这个视频**

# **什么是独石？**

**![](img/3f845a1c612fb4d17e19cb2d3217e232.png)**

**把 Monolith 想象成这个巨大的蛋糕(不完全是，但你明白了)，它有一个特定的目的，但完成了同样的最终结果(带来快乐)——来源:[https://pix abay . com/vectors/生日蛋糕-蜡烛-糖衣-奶油-33087/](https://pixabay.com/vectors/birthday-cake-candles-icing-cream-33087/)**

**monolith 是作为单个可部署单元构建的应用程序。示例包括单个 [Java EE 应用程序](https://wikipedia.org/wiki/Java_EE_application)或单个。NET Web 应用程序。单一应用程序通常与数据库和客户端用户界面相关联。**

# **什么是微服务？**

**![](img/cba90edbaae76ea198998fcdbfcb12ee.png)**

**把微服务想象成纸杯蛋糕(不完全是，但你明白了)，它服务于一个特定的目的，但完成了同样的最终结果(带来快乐)——来源:[https://pix abay . com/illustrations/cupcakes-wallpaper-paper-background-2887270/](https://pixabay.com/illustrations/cupcakes-wallpaper-paper-background-2887270/)**

**为适应应用程序功能而构建的单一服务。在微服务模式中，应用程序是多个服务的集合，每个服务都有特定的目标。例如，您可能有一个为客户处理购物车的服务，另一个处理付款服务，还有一个用于与股票的后端应用程序接口。这些微服务应该是松散耦合的，它们应该通过定义良好的 API 相互连接。它们可以用不同的语言和框架编写，可以有不同的生命周期。**

# **为什么要转向基于微服务的架构？**

**假设您是一名 IT 专业人员，负责一个需要现代化的复杂网站，该网站被设计成一个整体，现在是时候迁移到云了。也许已经完成的事情对你来说是最有用的:但当需要升级服务或添加新功能时，你就开始遇到问题:比如功能的推出要花很长时间，因为你的整体应用程序每个季度只能部署一次。不仅如此，以下是转向微服务的一些原因:**

*   **微服务可以独立测试和部署。部署单元越小，部署就越容易。**
*   **它们可以在不同的语言和框架中实现。对于每项微服务，您都可以自由选择最适合其特定用例的技术。**
*   **他们可以由不同的团队管理。微服务之间的边界使得一个团队更容易专注于一个或几个微服务。**
*   **通过迁移到微服务，您可以放松团队之间的依赖性。每个团队只需要关心他们所依赖的微服务的 API。团队不需要考虑那些微服务是如何实现的，不需要考虑它们的发布周期等等。**
*   **你可以更容易地为失败而设计。通过明确服务之间的界限，可以更容易地确定服务关闭时该做什么。**

# **如何迁移到基于微服务的架构？**

**没有良方，很难确切知道如何将您的应用过渡到微服务，从而最大限度地减少风险、停机时间、客户不满和资金损失。通过向云迁移和转向微服务来实现平台的现代化需要学习新的工具和新的工作方式。所有这一切看起来让人不知所措。但是这些担心不应该阻止你从基于微服务的架构中获益。**

## **迁移模式**

**如果你只是一次迁移所有东西，会有很多风险。但是如果你一个功能一个功能的迁移，你可以避免大量的风险。这种向云的迁移有三种主要模式:**

*   ****提升和转移:**这意味着您实际上只是将相同的工作负载或应用程序从一个位置转移到另一个位置。没有改变，没有编辑，只是按原样移动。**
*   ****改进和迁移:**这是一种重新利用现有云解决方案的模式。例如，通过事件处理进行同步到异步流的更改，或者在某些情况下甚至从整体服务器迁移到微服务器。**
*   ****推倒重来:**这包括使用云原生组件对应用程序进行彻底的重新设计。示例:使用托管和无服务器服务。**

**我们将使用一种特定风格的“推倒重来”模式，这种模式被逐渐应用到应用程序的每个特性上，而不是应用到整个应用程序上。**

**在接下来的几篇文章中，我们将使用电子商务网站作为例子，因为其中许多网站都是用单一的专有工具构建的，所以它们确实是这种迁移的良好候选。但是，即使您没有电子商务网站，您也可以将我们将在这里介绍的原则应用于广泛的工作负载。**

**除了这种迁移模式，我们还将向您展示两种混合模式。在迁移过程中，应用程序有一个混合架构，其中一些功能位于云中，一些功能仍在内部。迁移完成后，整个应用程序托管在云中，但它仍然与本地的后端服务进行交互。**

# **结论**

**如果您希望将现有的单一平台迁移到云中，那么您已经初步了解了本系列剩余部分的内容。更多文章敬请关注[云烹饪系列](/@pvergadia/get-cooking-in-cloud-an-introduction-5b3b90de534e)。**

# **后续步骤和参考:**

*   **在[谷歌云平台媒体](https://medium.com/google-cloud)上关注这个博客系列。**
*   **参考:[整体到 GKE 解决方案](https://cloud.google.com/solutions/migrating-a-monolithic-app-to-microservices-gke)的微服务。**
*   **Codelab: [将一个整体迁移到 GKE 的微服务上](https://codelabs.developers.google.com/codelabs/cloud-monolith-to-microservices-gke/#0)**
*   **关注[获取云端烹饪](https://www.youtube.com/watch?v=pxp7uYUjH_M)视频系列，订阅谷歌云平台 YouTube 频道**
*   **想要更多的故事？查看我的[媒体](/@pvergadia/)，[在 twitter 上关注我](https://twitter.com/pvergadia)。**
*   **请和我们一起欣赏这部迷你剧，并了解更多类似的谷歌云解决方案:)**