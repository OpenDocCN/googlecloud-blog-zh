# 在谷歌云平台上实现#无服务器

> 原文：<https://medium.com/google-cloud/going-serverless-on-google-cloud-platform-b770803c5ee7?source=collection_archive---------0----------------------->

现在叫无服务器的东西很热门。每个开发工具公司都发现了这一点，突然间一切都变得不需要服务器了。

事实上，在谷歌云未来 17 个主题演讲中，很多东西都被称为无服务器。这些产品本身就令人惊叹，但是“无服务器”这个词被随意使用，以至于有时令人生厌。

不出所料，[人对这个](/@cloud_opinion/googles-folly-d57166b40045)感到不安。

那么无服务器到底是什么？还是那句话，什么是无服务器，每个人都有自己的看法。

所以也许只有凯尔西关心这个😊

让我来定义一下*我的*定义。

我将把“云”产品分为四个不同的类别。名字基本上都是编出来的，但这是我觉得有意义的。

## 1) SaaS

最终用户可以直接使用 SaaS 的产品。没有要供应的东西。你要么为你所使用的付费，要么购买某种批量信用。

## 2)无服务器

开发人员使用无服务器产品来构建高阶产品。没有什么可供应的，但是可能有一些旋钮来调整基础设施。您只需为您使用的内容付费，仅此而已。

## 3)管理

开发人员使用托管产品来构建高阶产品。开发人员需要提供容量和进行调整，但不需要维护基础设施。当基础设施未被使用时，您需要为多余的基础设施付费。

## 4) IaaS

IaaS 产品反映了内部解决方案，只是您可以在几秒钟/几分钟而不是几天/几个月内调配资源。它们需要明确的供应和维护。当基础设施未被使用时，您需要为多余的基础设施付费。

你同意这种分类吗？我是不是在各方面都完全错了！！？
在评论里告诉我吧！

我将把谷歌云的大部分产品归入这四个类别中的一个。当然，可以提出让产品适合多个类别的论点，但我将做出艰难的决定，只把它们放在一个类别中。我想这会让我很清楚我是如何看待这些类别的。

# 1) SaaS

[**G 组曲**](https://gsuite.google.com/)
明显的进入了这一类。G Suite 可供最终用户使用，不需要开发人员做任何事情。当然，Apps 脚本可以被认为是无服务器的，但是由于它如此依赖于 G Suite 的其余部分，它在这一类别中感觉是正确的。

[**Data Studio**](https://www.google.com/analytics/data-studio/)
Data Studio 让你为你的 Google Analytics、云 SQL、BigQuery 等数据建立仪表盘。这涉及到许多拖放操作，最终用户可以在没有开发人员支持的情况下高效工作。

# 2)无服务器

[**云功能**](https://cloud.google.com/functions)
很明显的进入了这个范畴。AWS Lambda 开启了 FaaS 革命，云功能也遵循类似的模式。您编写函数，指定它们触发的时间，调整每次调用分配多少内存/cpu，然后部署。其他的都是为你处理的。您为每个函数调用付费，因此它与您的流量完全匹配。

[**Cloud Run**](https://cloud.google.com/run)
Google 的无服务器容器产品，Cloud Run 类似于云功能或 App Engine，只是你部署了一个容器而不是代码。其他的都是为你处理的。云对每个传入的请求收费，所以你只需为你所使用的付费。

你也可以使用在 GKE 上运行的云，在这种情况下，它变得比无服务器更易于管理。更多详情见 GKE 部分。

[**云存储**](https://cloud.google.com/storage) 云存储类似于 Google Drive，但最终用户无法使用。它需要开发者在上面构建更高阶的系统。您不需要提供任何东西，只需为您使用的东西付费。您可以通过选择存储桶类型(近线、多区域等)来调整延迟和价格。

[**App 引擎标准**](https://cloud.google.com/appengine/docs/standard/) 首款 Google 云产品。编写您的代码，调整您想要的内存/cpu 数量，并进行部署。其他的都是为你处理的。App Engine Standard 可扩展至与您的流量完全匹配，当您没有流量时可扩展至零，因此您只需为您使用的流量付费。

[**云数据存储**](https://cloud.google.com/datastore) 首个 NoSQL 数据库亮相 Google Cloud。您不需要配置任何东西，只需要读写数据。您需要为每次读/写和您使用的确切存储付费，它会自动扩展。

[**Firebase 实时数据库/托管**](http://firebase.google.com) 类似于云数据存储，但是直接在前端使用。您不需要配置任何东西，只需要读写数据。您为数据传输和您使用的确切存储空间付费，它会自动扩展。

[**Cloud Firestore**](http://firebase.google.com) 这款下一代数据库结合了数据存储和 Firebase 实时数据库的优点。您可以从前端(如实时数据库)和服务器(如数据存储)直接使用它，并获得高级查询支持和强大的一致性。您不需要配置任何东西，只需要读写数据。您需要为每次读/写和您使用的确切存储付费，它会自动扩展。

[**big query**](https://cloud.google.com/bigquery/)big query 是 Google 的数据仓库服务，让用户用 SQL 查询 TB 的数据。没有什么需要管理的，只是转储/流式传输您的数据并进行查询。您只需支付存储数据和查询数据的费用，无需调配机器或存储或建立索引。

[**Pub/Sub**](https://cloud.google.com/pubsub/)Pub/Sub 是 Google 的多对多异步消息总线(想想 Apache Kafka 或者 RabbitMQ)。不需要提供任何东西，它可以瞬间扩展到每秒数百万条消息。Pub/Sub 对每条消息收费，因此您需要为您使用的内容付费。

[**【stack driver(日志、监控、调试、跟踪)**](https://cloud.google.com/stackdriver)
stack driver 工具让你无需任何设置就能访问强大的工具。他们都有一个免费层，但你可以付费来监控更多的资源和其他云，如 AWS。不需要配置或调整任何东西，而且您需要为监控的每个资源付费，因此没有额外的支出。

[**云数据流**](https://cloud.google.com/dataflow/)
云数据流使用 Apache Beam 创建完全托管的 ETL、批处理和流分析管道。它会自动缩放来处理您的管道，并在没有剩余工作时缩放回零。

## 3)管理

[**Kubernetes 引擎**](https://cloud.google.com/container-engine/)Google Kubernetes 引擎为你一键创建一个 Kubernetes 集群。Google 管理主节点和节点，并为您自动更新它们。您必须提前配置一个集群(尽管有一些自动扩展功能)，因此您要为未使用的资源付费。

[**App Engine Flexible**](https://cloud.google.com/appengine/docs/flexible/)App Engine Flexible 类似于 Standard，但是运行在虚拟机上而不是沙箱上。虽然这给了它更多的“灵活性”,但是你失去了标准的“无服务器”魔力。自动扩展和部署没有那么快，但最大的区别是您必须始终运行一个虚拟机实例，这意味着您要为未使用的资源付费。

[**云 SQL**](https://cloud.google.com/sql) 云 SQL 给你托管 MySQL 和 PostgreSQL 实例。Google 担心备份、故障转移复制等，这减少了运行数据库的操作开销。虽然有一些自动扩展功能，但您需要提前配置机器和磁盘，这意味着为未使用的资源付费。

[**云 Bigtable**](https://cloud.google.com/bigtable)云 Bigtable 是 Google 的 HBase 兼容 NoSQL 数据库。虽然存储会自动扩展(您可以选择 SSD 或 HDD)，但您需要选择想要调整性能的节点数量。更多节点=更高性能，但当然您可以过度/不足配置。

[**云扳手**](/google-cloud/cloud.google.com/spanner) 云扳手类似于 BigTable，除了它是全局一致和关系的，而不是 NoSQL。这太神奇了！您需要选择想要调优性能的节点数量。更多节点=更高性能，但当然您可以过度/不足配置。

[**云负载平衡(L4/L7)**](https://cloud.google.com/load-balancing) Google Cloud 提供的 L4 和 L7 负载平衡器都是完全托管的服务，不需要配置、预热或调整。它不属于无服务器类别的唯一原因是，无论负载平衡器处理多少流量，您都必须支付统一的费用来启动它。

[**Cloud data proc**](https://cloud.google.com/dataproc) Cloud data proc 启动托管 Spark/Hadoop 集群。您需要指定想要多少个虚拟机，并对它们进行调优，但之后您就可以使用集群，无需任何额外的设置。即使虚拟机未被使用，您也要为其付费。

[**云机器学习引擎**](https://cloud.google.com/ml-engine/) 云机器学习引擎创建并管理一个分布式 TensorFlow 集群，供你在上面训练和服务你的模型。您必须调配一个群集，但该群集是完全受管理的，您可以向它提交作业。

## 4) IaaS

[**计算引擎**](https://cloud.google.com/compute)这些是 Linux 和 Windows 虚拟机。虽然您可以使用托管实例组自动扩展它们，并且您可以做一些很酷的事情，比如让磁盘自动增长，但是最终您要 100%负责。

[**云启动器**](https://cloud.google.com/launcher)
云启动器让您在计算引擎上部署预配置的应用。虽然初始设置是自动的，但是您要负责维护服务器并为未使用的资源付费。

你怎么想呢?这有道理吗？这整个辩论值得吗？我错过了什么吗？请让我知道！