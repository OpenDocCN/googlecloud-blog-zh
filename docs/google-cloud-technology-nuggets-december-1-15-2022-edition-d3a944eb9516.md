# 谷歌云技术金块—2022 年 12 月 1 日至 15 日版

> 原文：<https://medium.com/google-cloud/google-cloud-technology-nuggets-december-1-15-2022-edition-d3a944eb9516?source=collection_archive---------3----------------------->

欢迎参加 2022 年 12 月 1 日至 15 日的谷歌云技术金块。

# **关于它将如何变化的预测**

预测#1:到 2025 年，90%的安全运营工作流将实现自动化，并作为代码进行管理。

预测#2:到 2025 年，五分之四的企业开发者将使用某种形式的管理开源

这些是我们的 IT 领导在 Google Cloud Next’22 上对未来几年可能发生的变化所做的一些预测。查看它们以了解更多:[预测#1](https://cloud.google.com/blog/products/identity-security/it-prediction-vast-majority-of-security-operations-workloads-will-be-automated) 和[预测#2](https://cloud.google.com/blog/products/open-source/predicting-the-rise-of-curated-open-source) 。

# **应用现代化**

API 管理和服务网格结合使用可以帮助组织简化向内部和外部用户公开微服务的方式。同时，通过针对其 API 的完整生命周期管理和性能分析，获得对其运营的可见性。查看这篇[文章](https://cloud.google.com/blog/products/application-modernization/api-management-and-service-mesh-go-together)解释了这两者是如何结合在一起的。

![](img/a3238f97a91a325cb1f2f29f79e7b8d7.png)

# **库伯内特和 GKE**

如果你有跨项目的后端，什么是最好的方式将这些暴露在互联网上，没有多个谷歌云全球负载均衡器(GLBC)的开销。本[指南](https://cloud.google.com/blog/topics/developers-practitioners/centralized-multi-cluster-ingress-anthos-service-mesh)以使用多集群入口和 Anthos 服务网格的形式提供了一个简洁的解决方案。

![](img/ece712aaf2e08c2a2408fd7e6dc2630c.png)

如果您在整个组织中管理集群，您如何确保他们遵循集群的最佳实践。现在可以通过名为 GKE 政策自动化的开源工具来获得帮助，该工具在谷歌云服务的基础上部署了一个解决方案。该工具可审计整个组织的集群，可一次性运行或定期运行，并与安全指挥中心集成。查看[博客文章](https://cloud.google.com/blog/topics/developers-practitioners/auditing-gke-clusters-across-entire-organization)了解更多详情。

![](img/c0344afd3c90588931200d153d9e9be6.png)

# **创业公司**

对于初创公司来说，时间至关重要。团队希望尽可能快地迭代。无服务器和完全托管的服务大大减少了开发人员和设计解决方案的工作量。查看这个[指南](https://cloud.google.com/blog/topics/startups/use-serverless-tech-to-iterate-quickly-in-a-startup-environment)，帮助你理解谷歌云如何帮助初创公司快速迭代。本文涵盖了您可以在哪里运行代码，我们的 IDE 扩展云代码有助于开发的内部循环，服务有助于您在自己的服务之间进行编排，甚至您可以在哪里使用自己的定制/批处理代码。

![](img/e59c978739c03835e02b11f9da5532d3.png)

# **存储、数据库和数据分析**

AlloyDB for PostgreSQL 是一个完全托管的、兼容 PostgreSQL 的数据库服务，现已正式推出。它与 Google 的最佳产品 PostgreSQL 完全兼容:横向扩展计算和存储、集成分析和 AI/ML 驱动的管理。

自今年 5 月发布预览版以来，已经增加了几个功能。其中包括:客户管理的加密密钥(CMEK)和 VPC 服务控制等安全功能、跨区域复制预览和其他配置选项。查看[的博客文章](https://cloud.google.com/blog/products/databases/announcing-the-general-availability-of-alloydb-for-postgresql)，了解更多关于 AlloyDB、客户动力等的详细信息。

![](img/453be97cb7725cc79dbdb16b0fefaad1.png)

如果您一直在独自管理云中的 SQL 数据库，我们可以直接从客户那里获得一些数据，了解他们使用 Cloud SQL 的体验，Cloud SQL 是我们完全托管的 SQL 数据库产品。[白皮书](https://cloud.google.com/blog/products/databases/the-business-value-of-cloud-sql)直接获取客户提供的信息。

![](img/2746035d9ff36c9661790d4c0d28f428.png)

如果您是一个云发布/订阅用户，您可能面临的挑战之一是您必须处理自己的处理逻辑以避免重复消息。现在，这个问题可以通过 Cloud Pub/Sub 中现在普遍提供的恰好一次交付功能来解决。查看[帖子](https://cloud.google.com/blog/products/data-analytics/cloud-pub-sub-exactly-once-delivery-feature-is-now-ga)，它提供了一篇有趣的文章，介绍了这一特性的好处以及它是如何实现的。

![](img/54d8a9589d6322ac07922cef5e02faa9.png)

继续 Pub/Sub 主题，Pub/Sub Group Kafka 连接器现已全面上市。这将允许您保持您的 Google 云系统和基于 Kafka 的系统同步。连接器中提供了源和接收器连接选项。查看[帖子](https://cloud.google.com/blog/products/data-analytics/pubsub-group-kafka-connector-is-now-ga)了解更多详情。

将大型数据集导入 BigQuery 时，最好使用哪种文件格式？这些文件格式应该压缩还是解压缩？看看这篇[的博客文章](https://cloud.google.com/blog/products/data-analytics/performance-considerations-for-loading-data-into-bigquery)，它在多种文件格式(CSV、AVRO、拼花等)之间展开了一场较量，并展示了哪一种是最有效的。

![](img/a65910770d76515e4a01eb657e1dd776.png)

你听说过这一年来使 BigQuery SQL 更加用户友好的特性吗？我确信在这篇[博文](https://cloud.google.com/blog/topics/developers-practitioners/year-review-bigquery-user-friendly-sql)中会有一两个让你惊讶的特色。

# **机器学习**

今年早些时候发布的谷歌翻译中心受到了客户的好评。查看这篇[博客文章](https://cloud.google.com/blog/products/ai-machine-learning/a-closer-look-at-translation-hub)，它将帮助您了解 Google 翻译中心的各种功能，尤其是针对企业的功能，以及他们是如何利用这些功能的。

![](img/c10a716732ea1ba5f53a2f969f80d856.png)

连续评估 ML 模型的质量的过程可能是困难且昂贵的任务。顶点人工智能模型评估使您能够迭代地评估和比较大规模的模型性能。使用 Vertex AI 模型评估，您可以定义一个测试数据集、一个模型和一个评估配置作为输入，无论您是使用笔记本电脑训练模型、运行训练作业还是在 Vertex AI 上运行 ML 管道，它都会返回模型性能指标。查看这篇[博客文章](https://cloud.google.com/blog/topics/developers-practitioners/improving-model-quality-scale-vertex-ai-model-evaluation)了解更多信息。

![](img/5815f7b4e265d893a63cafac5e3e6606.png)

继续类似的话题，如果你正在寻求通过分布式训练方法来改善基于 PyTorch 的深度学习模型的训练过程，请查看这篇[帖子](https://cloud.google.com/blog/products/ai-machine-learning/efficient-pytorch-training-with-vertex-ai)。

![](img/9e7679ae915cc4441aa26bfa45fb4541.png)

# **原料药**

你如何设计你的 API，使其可读性更强，更简洁，更灵活。查看 RESTful web API 设计中要避免的 6 个常见错误。

![](img/5305baca251b3976917524b2cbb62fc1.png)

API 安全性对于任何组织来说都是至关重要的。随着公共 API 不断受到威胁，如何设计 API 安全策略？看看这个[的帖子](https://cloud.google.com/blog/products/identity-security/build-your-api-security-strategy-on-these-4-pillars)。

# **开发者和从业者**

如果你已经使用谷歌云有一段时间了，你可能已经处理过计费提醒，当一个月的费用超过一定的阈值时，你会收到通知。如何通过自动化接收这些预算警报的流程，然后采取一些措施(比如停止虚拟机)来控制您的成本？).查看这篇[文章](https://cloud.google.com/blog/topics/developers-practitioners/using-budgets-automate-cost-controls)，它部署了一个名为 Cost Sentry 的 DeployStack 解决方案，该解决方案使用 Google 云服务为您建立了一个解决方案，您可以进一步定制该解决方案，以在预算超过特定阈值时实施您想要采取的行动。

![](img/f9b3b14f2c541de76302f2b6d01f1203.png)

Cloud Run 目前有一个运行作业的预览功能(Cloud Run Jobs)。作业是一个长时间运行的过程，它运行自己的任务并在完成后退出。它不侦听或服务请求。你能在工作中发挥什么样的功能？嗯，数据库迁移或导出就是这样一个任务，您可以在云运行作业中抽象出来。查看这篇[文章](https://cloud.google.com/blog/topics/developers-practitioners/running-database-migrations-cloud-run-jobs)，它涵盖了使用云运行作业运行数据库迁移。

查看关于工作流模式和最佳实践系列的最后一部分[，其中作者介绍了工作流生命周期和在工作流中使用 Firestore 的好处。也检查我们系列的](https://cloud.google.com/blog/topics/developers-practitioners/workflows-patterns-and-best-practices-part-3)[第 1 部分](https://cloud.google.com/blog/topics/developers-practitioners/workflows-patterns-and-best-practices-part-1)和[第 2 部分](https://cloud.google.com/blog/topics/developers-practitioners/workflows-patterns-and-best-practices-part-2)。

![](img/276b72d9f3510500f9ffb764aada765a.png)

如果你正在寻找一个有趣的应用程序来构建这个月，那么预测一部电影的分数并在这个过程中了解 Vertex AI、BigQuery 和 MongoDB Atlas 怎么样？查看这篇[文章](https://cloud.google.com/blog/topics/developers-practitioners/movie-score-prediction-bigquery-vertex-ai-and-mongodb-atlas)，它给出了如何构建这个应用程序的一步一步的过程。

![](img/f1fef96838bf0af6fd2fc651d36f481f.png)

# **了解谷歌云**

十二月是一个好好休息并在新的一年重新开始的时间。如果你要休息一段时间，而谷歌云已经在你的学习范围之内，尝试多种途径来学习更多。这些活动包括动手实验室、网络研讨会、竞赛等。查看指南，该指南提供了 [12 种在假期学习谷歌云的免费方法](https://cloud.google.com/blog/topics/training-certifications/cloud-skills-training-for-hot-tech-careers)。

![](img/f13763a21f355fc7de9123f1c84d4e4f.png)

Apigee 是 Google Cloud 自带的 API 管理服务。虽然您可能熟悉一般的 API 术语，但 Apigee 有时有自己的定义。看看这个[指南](https://cloud.google.com/blog/products/api-management/understanding-commonly-used-apigee-terms)，帮助你揭开常用术语的神秘面纱，

# 保持联系

对这份时事通讯有任何问题、意见或其他反馈吗？请发送[反馈](https://forms.gle/UAsAS7YLxYSBTNBy9)。

想要关注新的谷歌云产品发布吗？我们有一个方便的页面，您可以将它加入书签→[Google Cloud](https://bit.ly/3umz3cA?utm_source=ext&utm_medium=partner&utm_campaign=CDR_rom_gcp_gcptechnuggets_feb-a-2022_021622&utm_content=-)的新功能。