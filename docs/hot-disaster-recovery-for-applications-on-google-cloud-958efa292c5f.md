# Google Cloud 上应用程序的热灾难恢复

> 原文：<https://medium.com/google-cloud/hot-disaster-recovery-for-applications-on-google-cloud-958efa292c5f?source=collection_archive---------2----------------------->

## 在云中烹饪

![](img/e71c320d9e819915d2cb7d747336f104.png)

# 介绍

*[***云上做饭***](/@pvergadia/get-cooking-in-cloud-an-introduction-5b3b90de534e)*是一个[博客](/@pvergadia/get-cooking-in-cloud-an-introduction-5b3b90de534e)和[视频](https://www.youtube.com/playlist?list=PLIivdWyY5sqIOyeovvRapCjXCZykZMLAe)系列，帮助企业和开发者在 Google Cloud 上构建商业解决方案。在这第二个迷你系列中，我将介绍 Google Cloud 上的灾难恢复。当你在网上时，灾难可能很难处理。在这些文章中，我们将详细阐述如何应对地震、停电、洪水、火灾等灾害。**

**这是这个系列的计划。**

1.  **[灾难恢复概述](/google-cloud/hosting-web-applications-on-google-cloud-an-overview-87d0962931a3)**
2.  **[基于谷歌云的本地应用冷灾难恢复](/@pvergadia/cold-disaster-recovery-on-google-cloud-for-applications-running-on-premises-114b31933d02)**
3.  **[针对内部部署应用程序的谷歌云热灾难恢复](/google-cloud/warm-disaster-recovery-on-google-cloud-for-applications-running-on-premises-7428b0f7db72)**
4.  **[基于 Google Cloud 的内部应用热灾难恢复](/google-cloud/hot-disaster-recovery-on-google-cloud-for-applications-running-on-premises-da7048d1a57b)**
5.  **[谷歌云中应用的冷灾难恢复](/google-cloud/cold-disaster-recovery-for-applications-in-google-cloud-5edeb32f2fc6)**
6.  **[谷歌云中应用的温灾恢复](/google-cloud/warm-disaster-recovery-for-applications-in-google-cloud-9165b4ea8e2f)**
7.  **Google Cloud 中应用的热灾难恢复(**本文**)**
8.  **Google Cloud 上的数据灾难恢复:第 1 部分**
9.  **Google Cloud 上的数据灾难恢复:第 2 部分**

**在本文中，您将学习为部署在 Google Cloud 上的应用程序设置一个热灾难恢复模式。所以，继续读下去吧！**

# **你会学到什么**

*   **Google 云应用程序的热灾难恢复模式，带示例**
*   **灾难来袭前需要采取的步骤**
*   **发生灾难时需要采取的步骤**
*   **灾难发生后需要采取的步骤**

# **先决条件**

*   **谷歌云的基本概念和结构，这样你就可以识别产品的名称。**
*   **阅读[概述文章](/google-cloud/hosting-web-applications-on-google-cloud-an-overview-87d0962931a3)了解灾难恢复相关定义。**

# **看看这个视频**

**Google 云上的热灾难恢复**

# **让我们通过一个例子来学习 Hot DR 模式**

**我们在[的上一篇文章](/google-cloud/warm-disaster-recovery-for-applications-in-google-cloud-9165b4ea8e2f)中见过**鬃毛街头艺术**，但现在他们已经变得非常受欢迎，并获得了巨大的全球客户群，这要求他们必须始终在线。因此，我们将帮助他们建立一个几乎没有停机时间的热灾难恢复模式。**

**![](img/e7a9618013f18234f29123886fb06dc7.png)**

**曼诺街头艺术走向全球**

**Mane-street-art 有一个独特的优势，因为他们在 Google cloud 上部署他们的生产应用程序，他们可以充分利用内置的*高可用性* (HA)特性。**

***注意:如果您不熟悉这里使用的术语(RTO、RPO、灾难恢复模式),请查看之前的博客以获得概述。***

**在任何灾难恢复模式中，您都需要了解在灾难发生之前、之中和之后需要采取什么步骤。**

# **热灾难恢复模式—它是如何工作的？**

## **灾难来袭前应采取的步骤**

*   **他们必须建立一个 VPC 网络**
*   **创建使用应用程序服务配置的自定义图像。**
*   **使用该图像创建一个实例模板。**

**![](img/dfc3080bfeb659e515b8c01dd977daaa.png)**

**热灾难恢复模式:灾难来袭前应采取的步骤**

*   **使用此实例模板，配置一个区域托管实例组。这是为了确保弹性，如果整个区域关闭，实例在其他区域仍然可用，以保持应用程序运行。**
*   **确保在受管实例组上配置了 HTTP 和实例运行状况检查，以验证服务和实例是否在组内正常运行。这确保了如果实例上的服务失败，该组会自动重新创建该实例。**
*   **使用跨三个区域的托管实例组配置负载平衡。**
*   **在一个区域中创建 CloudSQL 作为主区域，并在另一个区域中启用副本。这样做意味着在主服务器不可用的情况下，可以将副本服务器提升为主服务器。**
*   **最后，将云 DNS 配置为指向主应用程序。**

**![](img/5f63241ea456c26f07c6ce251169a3d9.png)**

**热灾难恢复模式:灾难来袭前应采取的步骤**

## **灾难来袭时应采取的措施**

> **当灾难降临时，街头艺术所要做的就是看着系统自我修复。**

## **灾难过去后应采取的步骤**

**由于我们构建了一个内置高可用性的自我修复系统，因此在灾难期间和之后没有太多事情可做。在灾难期间，实例在不同的区域根据需要来来去去。**

# **结论**

**如果您正在 Google Cloud 上部署应用程序，并且必须满足非常低的 RTO 和 RPO 值，如 Mane-Street-Art，那么请使用一些内置的 Google Cloud HA 功能。敬请关注即将发布的文章，在这些文章中，您将了解如何为数据设置更多灾难恢复。**

# **后续步骤**

*   **在[谷歌云平台媒体](https://medium.com/google-cloud)上关注这个博客系列。**
*   **参考[灾难恢复解决方案](https://cloud.google.com/solutions/dr-scenarios-planning-guide)。**
*   **关注[获取云端烹饪](https://www.youtube.com/watch?v=pxp7uYUjH_M)视频系列，订阅谷歌云平台 YouTube 频道**
*   **想要更多的故事？查看我的[媒体](/@pvergadia/)，[在 twitter 上关注我](https://twitter.com/pvergadia)。**
*   **请和我们一起欣赏这部迷你剧，并了解更多类似的谷歌云解决方案:)**