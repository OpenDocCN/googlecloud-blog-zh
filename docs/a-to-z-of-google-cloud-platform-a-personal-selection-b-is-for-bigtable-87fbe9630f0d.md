# Google 云平台的 a 到 Z a 个人选择— B 代表 BigTable

> 原文：<https://medium.com/google-cloud/a-to-z-of-google-cloud-platform-a-personal-selection-b-is-for-bigtable-87fbe9630f0d?source=collection_archive---------1----------------------->

现在我不打算详述 BigTable 本身，而是更多地讨论它与 [HBase](https://hbase.apache.org/) 的关系。为了弥补我破坏规则的条目，这是一个非常短的帖子。Hbase 基于谷歌的 [BigTable](http://research.google.com/archive/bigtable.html) ，GCP 提供 [Bigtable](https://cloud.google.com/bigtable/docs/) 作为托管产品。是的，我能听到成千上万的“但是锁定怎么办？”，“我在 Hbase 上投入了时间和精力”…。但奇怪的是，你可以在很大程度上忘记管理自己的 Hadoop 集群(这是 Hbase 的基础)，但可以让谷歌管理所有这些基础设施，因为你可以使用[云 Bigtable HBase 客户端](https://cloud.google.com/bigtable/docs/bigtable-and-hbase)。是的，我仍然能听到成千上万的哭声，是的，有些不同。但是列表很小，您需要决定是否值得花费时间和精力来管理您自己的 hadoop 集群！顺便说一下，如果你真的想要一个 hadoop 集群，那么 Google 在 [Dataproc](https://cloud.google.com/dataproc/overview) 中为你提供了一个托管 hadoop 和 spark 服务

题外话:本系列中的 M 主题太多，所以我不会谈论 Hadoop 生态系统基本上是如何从 Google 的 mapreduce 和 Google 文件系统论文中派生出来的。你可以自己阅读研究论文，可以在这里找到[这里](http://research.google.com/archive/mapreduce.html)和[这里](http://research.google.com/archive/gfs.html)。

我发现令人震惊的是，谷歌需要发明多少技术来满足互联网规模企业的需求，这些技术已经成为某种形式的开源项目，基本上变成了基础的乐高积木。