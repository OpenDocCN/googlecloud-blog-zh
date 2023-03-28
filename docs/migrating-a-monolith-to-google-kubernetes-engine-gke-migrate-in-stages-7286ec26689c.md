# 将一个整体迁移到谷歌 Kubernetes 引擎(GKE)——分阶段迁移

> 原文：<https://medium.com/google-cloud/migrating-a-monolith-to-google-kubernetes-engine-gke-migrate-in-stages-7286ec26689c?source=collection_archive---------2----------------------->

## 在云中烹饪

作者:[普里扬卡·韦尔加迪亚](https://twitter.com/pvergadia)，[卡特·摩根](https://twitter.com/carterthecomic)

![](img/c4bcbde49cdae22ad3e3f5c644ded927.png)

# 介绍

*[***获取云端烹饪***](/@pvergadia/get-cooking-in-cloud-an-introduction-5b3b90de534e)*是一个[博客](/@pvergadia/get-cooking-in-cloud-an-introduction-5b3b90de534e)和[视频](https://www.youtube.com/playlist?list=PLIivdWyY5sqIOyeovvRapCjXCZykZMLAe)系列，帮助企业和开发者在 Google Cloud 上构建商业解决方案。在这第三个迷你系列中，我们将讲述 ***将一个整体迁移到谷歌 Kubernetes 引擎(GKE)*** 。将整体迁移到微服务可能会令人生畏。一旦你决定接受它，你需要考虑什么？继续阅读…**

**在这些文章中，我们将带您了解将 monolith 迁移到微服务的整个过程、迁移流程、首先迁移什么、迁移的不同阶段以及如何处理数据迁移。我们这些文章的灵感来自于[这篇解决方案文章](https://cloud.google.com/solutions/migrating-a-monolithic-app-to-microservices-gke)。我们将以一个真实的客户故事来结束这一切，在一个真实的应用程序中完成这些步骤。**

**以下是这部迷你剧的所有文章，供你查阅。**

1.  **[将一块巨石迁移到 GKE:概述](/google-cloud/migrating-a-monolith-to-google-kubernetes-engine-an-overview-785f2cbe5c62)**
2.  **[将整块材料迁移到 GKE:迁移过程](/google-cloud/migrating-a-monolith-to-google-kubernetes-engine-gke-migration-process-2de2f51986a2)**
3.  **将一块巨石迁移到 GKE:分阶段迁移(本文)**
4.  **将一块巨石迁移到 GKE:先迁移什么？**
5.  **[将一个整体迁移到 GKE:数据迁移](/google-cloud/migrating-a-monolith-to-google-kubernetes-engine-gke-data-migration-ef8ebccef6b0)**
6.  **[将一个整体迁移到 GKE:客户故事](/google-cloud/migrating-a-monolith-to-google-kubernetes-engine-gke-customer-story-c35c320325eb)**

**在本文中，我们将详细介绍如何分阶段迁移一个特性。所以，继续读下去吧！**

# **你会学到什么**

*   **如何为迁移准备您的 Google 云环境**
*   **如何分析您计划迁移的服务的依赖性**
*   **如何将功能从您的传统环境迁移到新的基于云的微服务环境**

# **先决条件**

**开始之前，了解以下内容会有所帮助:**

*   **谷歌云的基本概念和结构，这样你就可以识别产品的名称。**
*   **本[之前的视频在云系列](/@pvergadia/get-cooking-in-cloud-an-introduction-5b3b90de534e)中获取烹饪。**

# **看看这个视频**

# **一个用例示例**

*****冰淇淋店，*** 是一家(虚拟的)网上冰淇淋店。他们正在将他们的 monolith 网站迁移到微服务，并需要将他们的第一个微服务迁移到 GKE 的指导。他们也有这个星球上最好的(想象中的)冰淇淋。**

**在他们可以迁移的所有功能中(查看下一篇文章了解如何确定)，他们决定将购物车从遗留的 monolith 应用程序迁移到 GKE。**

**总的来说，我们推荐 ***冰激凌理论*** 将网站的功能一个一个的迁移到新的环境中，在需要的时候创建微服务。这些微服务可以在需要时回调遗留系统。**

**像这样分阶段迁移有两个好处:**

1.  **较小的项目更容易处理，最大限度地减少错误和代价高昂的错误。**
2.  **它让您能够轻松处理每个单独的迁移。**

**让我们看看怎么做。**

# **首要任务:准备您的云环境**

**![](img/69e6f21ef8030795786d255ed1056473.png)**

****Mise en place** 是一个法国烹饪短语，意思是“放在适当的位置”或“一切都在适当的位置”，指的是在烹饪之前*需要的准备工作。对我们来说，这意味着在迁移特性之前，要确保我们要迁移到的环境设置正确。来源:[https://pix abay . com/illustrations/cooking-baking-recipe-book-chef-hat-4206076/](https://pixabay.com/illustrations/cooking-baking-recipe-book-chef-hat-4206076/)***

**在将功能转化为 GKE 的微服务之前，我们需要一个地方来放置它们。为此，我们必须准备好谷歌云环境。**

**首先要建立的是一个 Google Cloud [组织](https://cloud.google.com/resource-manager/docs/cloud-platform-resource-hierarchy#organizations)。**

**然后，我们需要确保只有拥有正确授权的人才能使用谷歌云[策略](https://cloud.google.com/iam/docs/policies)访问资源，以控制对谷歌云[资源](https://cloud.google.com/iam/docs/resource-hierarchy-access-control)的访问。**

**接下来，是时候创建一个计划来部署资源，以一种标准化的、可复制的方式，使用基础设施作为代码，利用像 [Deployment Manager](https://cloud.google.com/deployment-manager) 和 Terraform 这样的工具。**

**在这一点上，我们推荐 ***冰淇淋理论*** 团队研究 GKE 及其特色，以便他们可以根据自己的需求进行调整。这将意味着改变 GKE 的一些默认设置，并可能加强集群的安全性。**

**一旦建立了 GKE，就该使用像 Spinnaker、Jenkins 这样的工具或者像 Cloud Build 和 Container Registry 这样的 Google 云产品来构建 CI/CD 管道了。在项目早期这样做可以避免在生产中出现问题。**

**最后，我们必须决定不同的服务如何相互通信，以及基于 API 的解决方案(如 Apigee)还是私有连接(如 cloud interconnect)更适合它们的需求。**

**一旦我们弄清楚了托管环境，是时候开始迁移特性了。**

# **了解功能的依赖关系**

**![](img/10d0966da23e8d097be418cc2cc57728.png)**

**在烹饪(迁移功能)之前，理解食谱很重要。在这种情况下，我们必须意识到我们正在迁移的特性的依赖性。来源:[https://pix abay . com/vectors/recipe-label-icon-symbol-spoon-575434/](https://pixabay.com/vectors/recipe-label-icon-symbol-spoon-575434/)**

**首先要分析的是购物车功能对系统其余部分的依赖性。我们可以通过推理用户在与网站交互时采取的步骤来做到这一点:**

1.  **用户选择冰淇淋并点击“添加到我的购物车”。这触发了从用户浏览器到购物车的 API 调用**
2.  **一旦购物车收到调用，它就通过对库存系统进行 API 调用来检查所选的冰淇淋是否有货。**
3.  **如果商品有货，购物车将存储信息“Carter 在购物车中有一个草莓冰淇淋商品的实例。”**
4.  **最后，当用户结账并完成支付过程时，支付子系统查询购物车以计算总额。一旦支付完成，支付子系统通知购物车功能清空购物车。**

**因此，如果我们对此进行分析，从高层次来看，购物车功能有四个依赖项，它由前端和支付系统调用，并查询数据库和库存系统。**

**现在我们已经确定了依赖关系，让我们考虑一下为这个特性选择什么样的数据库。文档数据库非常适合存储购物车，因为购物车可以很容易地通过用户 id 进行索引，并且这里不需要关系数据库的强大功能。**

**Cloud Firestore 非常适合这一目的，它是一个托管的、无服务器的 NoSQL 文档数据库，所以让我们考虑将它作为我们的目标架构。**

# **将要素迁移到新环境**

**![](img/696a8f281df39497d9963db7f82734bb.png)**

**一旦我们迁移了这个特性，我们将能够享受到我们努力的美味回报。祝你好运！来源:[https://pix abay . com/vectors/food-plate-鱼-菜-虾-576547/](https://pixabay.com/vectors/food-plate-fish-vegetable-prawns-576547/)**

**现在最大的一块来了。迁移实际的购物车功能。为了简化讨论，让我们假设 ***冰淇淋理论*** 可以为他们的网站安排维护停机时间。**

1.  **首先，他们需要创建一个新的微服务来调用库存系统，并使用云 Firestore 来实现购物车 API。**
2.  **然后创建一个脚本，从遗留购物车系统中提取购物车条目，并将它们写入 Cloud Firestore。这个脚本需要以这样一种方式编写，它可以根据需要多次重新运行，但只复制自上次以来发生变化的购物车。**
3.  **然后创建一个相反的脚本，将购物车从 Cloud Firestore 复制回遗留子系统，以防他们需要回滚迁移。**
4.  **然后用 Apigee 公开购物车 API。**
5.  **他们需要准备和测试前端和支付系统的修改，以便这些系统可以调用新的购物车系统。**
6.  **又到了运行我们之前准备的数据迁移脚本的时候了，因此所有数据都是同步的，并且我们在这个过程中没有丢失任何购物车记录。**
7.  **此时，他们需要将网站置于维护模式。**
8.  **重新运行数据迁移脚本。**
9.  **将经过测试的修改部署到遗留生产系统的前端和支付系统。**

**一旦完成，他们准备禁用网站的维护模式，并通过他们的新微服务运行流量！**

# **结论**

**于是， ***冰激凌理论*** 网站的购物车功能现在是托管在谷歌云上的微服务。他们可以继续分发成吨的美味冰淇淋给他们快乐的顾客！**

**随着时间的推移，随着您的系统越来越多地基于微服务，整个系统变得比单一应用程序中的耦合更松散，迁移功能变得更容易，这些功能也更容易修改和部署。**

> **分阶段迁移有两个优点:较小的项目更容易处理，最大限度地减少错误和代价高昂的错误。它让您能够轻松处理每个单独的迁移。**

**如果你想将一个整体应用迁移到微服务，你已经通过 ***冰激凌理论*** 的用例领略了其中的步骤。敬请关注[云烹饪系列](/@pvergadia/get-cooking-in-cloud-an-introduction-5b3b90de534e)中的更多文章，并查看下面的参考资料了解更多细节。**

# **后续步骤和参考:**

*   **在谷歌云的媒介上关注这个博客系列。**
*   **参考:[GKE 解决方案](https://cloud.google.com/solutions/migrating-a-monolithic-app-to-microservices-gke)的整体到微服务。**
*   **Codelab: [将一个整体迁移到 GKE 的微服务上](https://codelabs.developers.google.com/codelabs/cloud-monolith-to-microservices-gke/#0)**
*   **关注[云烹饪](https://www.youtube.com/watch?v=pxp7uYUjH_M)视频系列，订阅谷歌云的 YouTube 频道**
*   **想要更多的故事？查看我的[媒体](/@pvergadia/)，[在 twitter 上关注我](https://twitter.com/pvergadia)。**
*   **请和我们一起欣赏这部迷你剧，并了解更多类似的谷歌云解决方案:)**