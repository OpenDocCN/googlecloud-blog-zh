# 使用 init 动作脚本在 Dataproc 上轻松部署 Trino

> 原文：<https://medium.com/google-cloud/easily-deploy-trino-on-dataproc-with-init-action-script-f23aef1a6cac?source=collection_archive---------2----------------------->

> 想在 Dataproc 上轻松部署 Trino 吗？你在搜索 Trino 初始化脚本吗？在这里。

**从 github 下载“**[**trino . sh**](https://github.com/sametkaradag/initialization-actions/blob/master/trino/trino.sh)**”**上传到你的 GCS 桶。

下面是我的 github 链接[https://github . com/sametkaradag/initial ization-actions/blob/master/trino/trino . sh](https://github.com/sametkaradag/initialization-actions/blob/master/trino/trino.sh)，它是[https://github . com/GoogleCloudDataproc/initial ization-actions](https://github.com/GoogleCloudDataproc/initialization-actions)的分支(在撰写本文时正在等待拉取)

如果您想使用 Trino 的 BigQuery 连接器来查询 BigQuery 数据，请用您的 project-id 替换 init 操作中的第 162 行:

```
bigquery.project-id=set-your-project-id
```

然后**创建您的 dataproc 集群**:

```
gcloud dataproc clusters create trino-test — enable-component-gateway — region europe-west4 \ — zone europe-west4-c — master-machine-type n1-standard-4 — master-boot-disk-size 100 — num-workers 8 \ — worker-machine-type n1-standard-4 — worker-boot-disk-size 100 — image-version 2.0-debian10 \ — scopes 'https://www.googleapis.com/auth/cloud-platform' — initialization-actions ‘gs://trino-init/trino.sh’ — project change-with-your-project-id
```

这里我使用 Trino 在短暂的 Dataproc 集群上进行 BigQuery 查询，这意味着我在处理之前创建集群，之后删除集群以降低成本。

我不会在 Dataproc 上存储任何数据，因此磁盘大小(worker-boot-disk-size，master-boot-disk-size)被设置为 100gb。

我在 n1-standard-4 机器上只使用了 2 个 worker 节点，这些机器有 15GB RAM。如果需要更快的查询，请增加这些值。

就是这样，现在你有了 Trino:)

最后，如何连接？

您可以使用 Trino CLI 客户端或 JDBC 客户端，如 [SquirrelSQL](http://squirrel-sql.sourceforge.net/) 、 [DBeaver](https://dbeaver.io/) (免费)或 [DataGrip](https://www.jetbrains.com/datagrip/download/#section=mac) (需要付费订阅)

您还可以配置您的 JDBC 客户端连接到 BigQuery，让一个客户端通过两个不同的会话来分析 BQ 数据。

如果你想看实际操作，这里有一个 [youtube 视频](https://youtu.be/n1PFnvfQbFE)和 match_recognize 演示。