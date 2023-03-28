# Google Cloud Dataproc 上临时集群的持久化应用程序历史

> 原文：<https://medium.com/google-cloud/persisting-application-history-from-ephemeral-clusters-on-google-cloud-dataproc-7b1b03a49686?source=collection_archive---------0----------------------->

因此，您希望使用短暂的 Dataproc 集群，但是不希望一旦您的集群被拆除，您就失去了宝贵的线索和火花历史。

进入[单节点 Dataproc 集群](https://cloud.google.com/dataproc/docs/concepts/configuring-clusters/single-node-clusters)。它将作为 MapReduce 和 Spark 作业历史的持久历史服务器，以及一个进入聚合纱线日志的窗口。

# 背景

已经有一段时间的 Hadoop 工程师可能已经习惯了应用程序历史的 YARN 和 Spark UIs