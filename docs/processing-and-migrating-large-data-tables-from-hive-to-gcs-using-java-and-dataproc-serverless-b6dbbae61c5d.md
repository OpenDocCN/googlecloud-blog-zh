# 使用 Java 和 Dataproc 无服务器处理大型数据表并将其从 Hive 迁移到 GCS

> 原文：<https://medium.com/google-cloud/processing-and-migrating-large-data-tables-from-hive-to-gcs-using-java-and-dataproc-serverless-b6dbbae61c5d?source=collection_archive---------1----------------------->

![](img/4b0cba1ab6b1039c0183a22d4d6c720f.png)

ataproc 是一个完全托管的云服务，用于在 Google 云平台上运行 Apache Spark 工作负载。它会创建临时集群，而不是为我们的所有作业配置和维护一个集群。

[Dataproc 模板](https://github.com/GoogleCloudPlatform/dataproc-templates/blob/main/README.md)提供了一种灵活易用的机制，用于在 Dataproc 无服务器上管理和执行用例，而无需开发它们。

这些模板实现了常见的 Spark 工作负载，让我们可以轻松地定制和运行它们。

# 目标

这篇博客文章阐述了如何使用 Dataproc Serverless 处理大量工作负载并将其从 [Apache Hive Metastore](https://docs.cloudera.com/runtime/7.2.1/hive-hms-overview/topics/hive-hms-introduction.html) 迁移到 Google cloud。

# 先决条件

为了运行这些模板，我们需要:

*   Google Cloud SDK 已安装并通过验证
*   启用了专用 Google 访问的 VPC 子网。默认子网是合适的，只要启用了私有 Google 访问。您可以在此查看所有 Dataproc 无服务器网络需求[。](https://cloud.google.com/dataproc-serverless/docs/concepts/network)

# 主要优势

*   使用 Dataproc Serverless 运行 Spark batch 工作负载，而无需提供和管理您自己的集群。
*   Hive to GCS 模板是开源的，完全可定制，并可用于简单的工作。
*   您可以从 Hive 摄取 AVRO、CSV、ORC 和 JSON 格式的数据到 GCS。

# 配置参数

模板中包含以下属性来配置执行—

*   `spark.sql.warehouse.dir=<warehouse-path>`:Spark SQL Hive 仓库的位置路径，Spark SQL 在该仓库中保存表。
*   `hive.gcs.output.path=<gcs-output-path>` : GCS 输入路径。
*   `hive.gcs.output.path=<gcs-output-path>` : GCS 输出路径。
*   `hive.input.table=<hive-input-table>` : Hive 输入表名。
*   `hive.input.db=<hive-output-db>` : Hive 输入数据库名称。
*   `hive.gcs.output.format=avro` : GCS 输出文件格式。这可以是 avro，csv，paraquet，json，orc。默认输出路径设置为 avro。
*   `hive.partition.col=<hive-partition-col>`:对配置单元数据进行分区的列名。
*   `hive.gcs.save.mode=overwrite`:将写入模式设置为 GCS。默认参数值是 overwrite。你可以在这里了解每种保存模式的表现[。](https://spark.apache.org/docs/latest/sql-data-sources-load-save-functions.html#save-modes)

“Hive to GCS”模板还有另外两个可选属性，用于在加载到 GCS 之前应用 spark sql 转换

```
--templateProperty hive.gcs.temp.table='temporary_view_name' 
--templateProperty hive.gcs.temp.query='select * from global_temp.temporary_view_name'
```

> **注意:**当使用转换属性时，Spark 临时视图的名称和查询中的表的名称应该完全匹配，以避免“table view not found”错误。

# 使用

1.  在 GCS 中创建一个暂存桶，以存储运行无服务器集群所需的依赖项。

```
export GCS_STAGING_BUCKET=”my-gcs-staging-bucket”
gsutil mb gs://$GCS_STAGING_BUCKET
```

2.克隆 Dataproc 模板库并导航到 Java 模板文件夹。

```
git clone [https://github.com/GoogleCloudPlatform/dataproc-templates.git](https://github.com/GoogleCloudPlatform/dataproc-templates.git)
cd dataproc-templates/java
```

3.通过导出提交所需的变量来配置 Dataproc 无服务器作业—

我们将使用提供的`bin/start.sh`脚本，并使用以下强制环境变量进行配置，以将作业提交给 dataproc 无服务器:

*   `GCP_PROJECT`:运行 Dataproc 无服务器的 GCP 项目 id。
*   `REGION`:运行 Dataproc 无服务器的区域。
*   `SUBNET`:Hive warehouse 所在的子网，以便在同一个子网中启动作业。
*   `GCS_STAGING_BUCKET` : GCS staging bucket 位置，Dataproc 将在此存储 staging 资产(应该在之前创建的 bucket 内)。

```
# GCP project id to run the dataproc serverless job
GCP_PROJECT=<gcp-project-id> # GCP region where the job needs to be submitted
REGION=<region># Subnet where the hive warehouse exists
SUBNET=<subnet># Staging storage location for Dataproc Serverless (Already done in step 1)
GCS_STAGING_LOCATION=<gcs-staging-bucket-folder>
```

4.执行配置单元到 GCS Dataproc 模板—

配置作业后，我们现在将触发`bin/start.sh`,指定我们想要运行的模板以及执行的参数值。

> 注意:提交作业时应该启用 Dataproc API。

```
bin/start.sh \
--properties=spark.hadoop.hive.metastore.uris=thrift://<hostname-or-ip>:9083 \
-- --template HIVETOGCS \
--templateProperty hive.input.table=<table> \
--templateProperty hive.input.db=<database> \
--templateProperty hive.gcs.output.path=<gcs-output-path>
```

5.监控和查看 Spark 批处理作业

在 [Dataproc Batches UI](https://console.cloud.google.com/dataproc/batches) 中提交作业后，您可以监控日志并查看指标。

# 参考

*   [Dataproc 无服务](https://cloud.google.com/dataproc-serverless/docs/overview)
*   [Dataproc 模板库](https://github.com/GoogleCloudPlatform/dataproc-templates)