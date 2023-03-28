# 从零到英雄—在 Google 云平台上运行 parse-server 第 0 部分—简介

> 原文：<https://medium.com/google-cloud/from-zero-to-hero-run-parse-server-on-google-cloud-platform-part-0-introduction-54e7725c7ba0?source=collection_archive---------1----------------------->

嗨，

这篇博客是关于如何在 GCP 上运行解析服务器的系列文章的第一部分。阅读完本系列的所有博客后，你将知道如何:

1.  通过 docker-compose 在本地机器上构建、运行和测试您的[解析服务器](https://github.com/parse-community/parse-server) nodejs+ MongoDB
2.  在低/大规模的**生产**中，在 [GCP 容器引擎](https://cloud.google.com/container-engine/) (Kubernetes)上部署、运行和测试您的[解析服务器](https://github.com/parse-community/parse-server) nodejs + MongoDB 实例
3.  集成 [kube-lego](https://github.com/jetstack/kube-lego) 为你的应用程序自动获取安全证书(通过加密 API)
4.  创建一个使用解析服务器 API 的简单移动(iOS)应用程序
5.  通过[解析 LiveQuery 协议](https://github.com/parse-community/parse-server/wiki/Parse-LiveQuery-Protocol-Specification)为您的移动应用添加**实时功能**

# 从零到英雄

在**从零到英雄**系列中我不会对你未来的专业知识做任何假设。我将尝试从头开始解释我将使用的每一项技术，因此如果您对一项或多项技术感到满意，请随意跳过理论部分，直接进入实践部分。我也会在博客中提到哪些部分是理论性的，哪些是实践性的。

# **动机**

在我发现自己在生产模式下运行应用程序时遇到了很多困难之后，我决定写这个系列的博客。在 google container engine 之前，我使用 google app engine + mongoLAB，它确实工作得很好，但后来我决定使用 Redis 实例来缓存一些常见记录，并使用 elastic-search 来创建一个快速和可扩展的搜索引擎，相信我，这只是创建大型应用程序时需要使用的一部分东西。天真的解决方案是要么为我想使用的每一项服务创建一个虚拟机，要么更好(也更昂贵)地使用一些**即服务**解决方案，如 Redis labs 和 elastic cloud。这就是我决定**停止**并重新思考什么是正确的方法的地方，所以我读了很多文章、博客，看了很多 youtube 视频，当时我唯一知道的是，我想只在 GCP 运行我的服务。我不想写我决定和 GCP 一起去的原因，但我可以说的是，我熟悉另外两大云提供商(AWS 和 AZURE ),并在我搬到 GCP 之前使用过其中一个。

感谢上帝，我找到了**谷歌容器引擎**，它满足了我所有的需求:

*   让我的服务彼此靠近
*   需要时快速添加更多服务
*   按需轻松/自动扩展服务
*   在 GCP 上运行我的所有服务

在我们动手之前，我想补充最后一点，我知道这个解决方案不是运行 parse-server 或云中任何其他 NodeJS 应用程序的最简单的解决方案，但我可以向你保证的是，这个解决方案非常灵活且具有成本效益，如果你的应用程序或解决方案具有很高的吸引力，你不需要重构所有的堆栈。

**请注意！**因为我是一个非常快乐的 mac 用户，所有的截图、视频、照片和其他资产将只与 **mac 用户相关**。如果你使用的是 windows 操作系统，你也可以运行它，除了本系列的第 4 部分，我将展示如何创建一个 iOS 应用程序。

那么，你准备好动手了吗？[点击此处阅读第 1 部分](/@ran.hassid/from-zero-to-hero-run-parse-server-on-google-cloud-platform-part-1-run-parse-server-and-mongodb-63a5f89f670d)