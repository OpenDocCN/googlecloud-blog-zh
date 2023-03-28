# 最小化 Java 库中的依赖痛苦

> 原文：<https://medium.com/google-cloud/minimize-dependency-pain-in-java-libraries-f071de45bdfb?source=collection_archive---------1----------------------->

谷歌云平台发布了从云资产到云网络安全的数十种产品的客户端库。这些都建立在很多常见的基础设施之上，包括 Guava、gRPC 和 protobufs，以及许多第三方开源项目，如 Netty 和 Jaxen。GCP 邻近的项目，如阿帕奇梁和春云 GCP 扩大范围和复杂性更进一步。

![](img/eb04e393d5c2f6bc335dc911129f9a38.png)

不同时区的不同团队和数百名开发人员构建了这些库。确保我们发布的最新版本的库能够无冲突地协同工作是一件大事。

通过我们从为客户开发和维护我们自己的库中学到的经验，我们已经开发了一组最佳实践，用于避免基于 Maven 材料清单(BOMs)和语义版本控制的 Java 库中的不兼容性。这些实践与任何发布其他 Java 项目所依赖的库的人相关。我们现在发布这些[Google Java 库最佳实践](https://jlbp.dev/):

*   [JLBP-1](https://jlbp.dev/JLBP-1.html) :最小化依赖性
*   [JLBP-2](https://jlbp.dev/JLBP-2.html) :最小化 API 面
*   JLBP-3 :使用语义版本控制
*   [JLBP-4](https://jlbp.dev/JLBP-4.html) :避免依赖不稳定的库和特性
*   [JLBP-5](https://jlbp.dev/JLBP-5.html) :避免与其他依赖项的类重叠的依赖项
*   [JLBP-6](https://jlbp.dev/JLBP-6.html) :一起重命名工件和包
*   [JLBP-7](https://jlbp.dev/JLBP-7.html) :让突破过渡变得容易
*   JLBP-8:将广泛使用的功能升级到稳定版本
*   [JLBP-9](https://jlbp.dev/JLBP-9.html) :支持您的消费者的最低 Java 版本
*   [JLBP-10](https://jlbp.dev/JLBP-10.html) :只要消费者需要，就保持 API 的稳定性
*   JLBP 11 号:与兼容的依赖项保持同步
*   [JLBP-12](https://jlbp.dev/JLBP-12.html) :明确支持水平和 API 稳定性
*   [JLBP-13](https://jlbp.dev/JLBP-13.html) :快速删除依赖关系中对已弃用功能的引用
*   [JLBP-14](https://jlbp.dev/JLBP-14.html) :为每个依赖项指定一个单一的、可覆盖的版本
*   [JLBP-15](https://jlbp.dev/JLBP-15.html) :为多模块项目制作 BOM
*   [JLBP-16](https://jlbp.dev/JLBP-16.html) :确保消费者的依赖关系的高版本一致性
*   [JLBP-17](https://jlbp.dev/JLBP-17.html) :协调重大变更的首次展示
*   [JLBP-18](https://jlbp.dev/JLBP-18.html) :只有在万不得已的情况下才遮蔽依赖
*   [JLBP-19](https://jlbp.dev/JLBP-19.html) :将每个包装放入一个模块中
*   [JLBP-20](https://jlbp.dev/JLBP-20.html) :给每个罐子一个模块名

这 20 个实践描述了如何组织一个库和它自己的依赖项，这样开发人员可以很容易地在他们自己的项目中采用它们，而没有冲突和链接错误。我将在这里总结三个最重要的实践:

# 最大限度地减少依赖性( [JLBP-1](https://jlbp.dev/JLBP-1.html)

每个库应该尽可能少地添加到它的从属类路径中，理想的情况是除了它自己什么也不添加。为困难和复杂的事情添加依赖项可能是一种可以接受的折衷，但是避免仅仅为了节省几行代码而添加依赖项。库的每个依赖项也是库的消费者的依赖项，增加了版本冲突和安全漏洞的风险。

其他 19 个最佳实践涵盖了减轻过度依赖影响的技术，但是没有什么比完全消除依赖更有效的了。

# 发布和消费物料清单( [JLBP-15](https://jlbp.dev/JLBP-15.html) )

有时大型依赖树是不可避免的；例如，当编写一个 Kubernetes 应用程序来集成多个 GCP 产品时，比如 BigQuery、Cloud Datastore、Cloud Translate 和 TensorFlow。您的项目需要为其中的每一项及其所有依赖项提供客户端库。

不要试图挑选这样一个项目所依赖的几十个工件的版本，[导入一个 BOM 并让它为你挑选版本](https://github.com/GoogleCloudPlatform/cloud-opensource-java/blob/master/DECLARING_DEPENDENCIES.md)。例如，**com . Google . Cloud:libraries-BOM**保证了 Google Cloud Java orbit 中所有工件的一致版本，这些工件协同工作，彼此不冲突。BOM 使得更新到新版本更加简单，因为你只需要更新一个工件，而不是几十个独立的和潜在冲突的工件。

当您发布自己的复杂、多模块库时，也要发布 BOM 并建议您的客户导入它。如果产品非常依赖于其他库，您可能也希望将这些库包含在 BOM 中。让依赖者在类路径中拥有一致的版本变得容易。

# 语义版本化是你的朋友( [JLBP-3](https://jlbp.dev/JLBP-3.html) )

一旦你发布了( [JLBP-8](https://jlbp.dev/JLBP-8.html) 、 [JLBP-10](https://jlbp.dev/JLBP-10.html) 和 [JLBP-12](https://jlbp.dev/JLBP-12.html) )，就要非常非常努力地避免引入不兼容性，但是当没有其他选择时，就增加主要版本。当你引入一个新特性时，删除次要版本。为其他所有东西添加补丁版本。[SEM ver 规范](https://semver.org/)列出了细节。

可供开发者选择的第三方库的广度和深度是 Java 生态系统的主要优势之一。然而，太多的选择会带来自身的问题。仔细考虑您依赖哪个库以及依赖多少个库。当你发布一个库的时候，注意不要增加消费者的依赖性问题。学习并遵循这 20 个最佳实践可以让你为你的用户提供一个顺利的、没有问题的途径来采用你的库。