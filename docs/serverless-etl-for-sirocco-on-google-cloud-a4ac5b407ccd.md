# Google Cloud 上 Sirocco 的无服务器 ETL

> 原文：<https://medium.com/google-cloud/serverless-etl-for-sirocco-on-google-cloud-a4ac5b407ccd?source=collection_archive---------1----------------------->

*本故事最初发表于*[*datan coff . ee*](https://datancoff.ee/2017/05/serverless-etl-for-sirocco-on-google-cloud/)*。*

我在 GCP 大数据博客上发表了一个 Sirocco 的无服务器 ETL 解决方案的架构。该解决方案利用了 Cloud Dataflow 的自动缩放功能，可以从云桶中的几篇新闻文章扩展到数据库中的数百万篇新闻文章。有了这个博客，你现在应该拥有了构建新闻监控或意见跟踪解决方案的所有组件。我知道这一点，因为我在一个实际的新闻监控解决方案中使用了完全相同的设置——在以后的帖子中会有更多的介绍。

我建议你这么做:

*   阅读 [Plutchik 的情感分析框架](/@datancoffee/opinion-analysis-of-text-using-plutchik-5119a80229ea)来理解这个解决方案背后的理论
*   阅读 [ETL 解决方案](https://cloud.google.com/blog/big-data/2017/05/designing-etl-architecture-for-a-cloud-native-data-warehouse-on-google-cloud-platform)
*   转到 [github repo](https://github.com/GoogleCloudPlatform/dataflow-opinion-analysis) 并按照自述文件中的说明进行操作。设置您自己的处理管道并运行一个测试，处理我上传到测试文件夹的一些新闻文章。