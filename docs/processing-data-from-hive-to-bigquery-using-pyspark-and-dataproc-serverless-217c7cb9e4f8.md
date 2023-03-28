# 使用 PySpark 和 Dataproc 无服务器处理从 Hive 到 BigQuery 的数据

> 原文：<https://medium.com/google-cloud/processing-data-from-hive-to-bigquery-using-pyspark-and-dataproc-serverless-217c7cb9e4f8?source=collection_archive---------1----------------------->

# 介绍

![](img/c1b94e02a76138a164a90ec60264e5bb.png)

Google Cloud Dataproc 徽标

## 目标

这篇博文解释了如何使用 PySpark、Google Cloud 上的 Dataproc Serverless 和 Dataproc 模板运行批处理工作负载，以处理从 Apache Hive 表到 BigQuery 表的数据。

两个使用案例涵盖如下:

1.  将您的 Hive 表和数据从 Hadoop 环境迁移到 Google Cloud。
2.  同时处理来自包括 Apache Hive 和 BigQuery 在内的数据源的数据。

## Dataproc 无服务器

正如[在 2021 年末宣布的](https://cloud.google.com/blog/products/data-analytics/spark-jobs-that-autoscale-and-made-seamless-for-all-data-users)，Dataproc Serverless 让我们能够[在 Google Cloud](https://cloud.google.com/hadoop-spark-migration) 上运行 Spark 工作负载，而不必担心集群管理和调优。

## Dataproc 模板

[Dataproc 模板](https://github.com/GoogleCloudPlatform/dataproc-templates)允许我们运行涵盖 Spark 工作负载常见用例的预建模板，并根据特定需求定制它们。他们使用 Dataproc Serverless 作为运行这些工作负载的执行环境。模板目前有 Java 和 Python 两种版本。

**在本帖**中，我们将讨论如何使用 PySpark **Hive 来 BigQuery 模板**。

花时间阅读这篇文章[也是一个好主意，这篇文章](/@ppaglilla/getting-started-with-dataproc-serverless-pyspark-templates-e32278a6a06e)是开始使用来自另一个 PySpark 用例的 Dataproc 模板的好参考:[**GCS to big query**](https://github.com/GoogleCloudPlatform/dataproc-templates/blob/main/python/dataproc_templates/gcs/README.md#gcs-to-bigquery)

## 配置单元到 BigQuery

执行时， [Hive to BigQuery](https://github.com/GoogleCloudPlatform/dataproc-templates/tree/main/python/dataproc_templates/hive) 模板将从您的 [Apache Hive](https://hive.apache.org/) 表中读取数据，并将其写入 [BigQuery](https://cloud.google.com/bigquery) 表。

# 先决条件

*   [Google Cloud SDK](https://cloud.google.com/sdk/docs/install) 安装并认证。你可以在 Google Cloud 控制台中使用 Cloud Shell，通过这个[链接](https://console.cloud.google.com/cloudshell/editor)来配置一个环境。
*   Python 3.7+已安装并添加到 PATH 变量中。
*   启用了专用 Google 访问的 VPC 子网。只要启用了私有 Google 访问，默认子网可能就足够了。您可以在这个[链接](https://cloud.google.com/dataproc-serverless/docs/concepts/network)查看 Dataproc 无服务器网络需求。

```
# Example updating default network to enable Private Google Access
gcloud compute networks subnets update default — region=us-central1 — enable-private-ip-google-access
```

# 使用

在 Dataproc Serverless 中提交 py spark 作业时，最好查看可能的[参数的文档。](https://cloud.google.com/sdk/gcloud/reference/dataproc/batches/submit/pyspark)

注意，我们将把 [Spark BigQuery 连接器](https://github.com/GoogleCloudDataproc/spark-bigquery-connector)作为作业提交中的. jar 依赖项传递，以允许 Spark 与 BigQuery 连接。

此处的所有代码片段都获得以下许可:

> *版权所有 2022 谷歌公司。*
> 
> *SPDX-许可证-标识符:Apache-2.0*

## 步伐

1.  首先，在 GCS 中创建一个 staging bucket，在 Dataproc Serverless 中提交一个 PySpark 作业，以便它存储我们的工作负载的依赖项。我们将使用 **gsutil mb** 命令创建一个存储桶:

```
export GCS_STAGING_LOCATION="gcs-staging-bucket-folder"gsutil mb
gs://$GCS_STAGING_LOCATION
```

**2。**克隆 Dataproc 模板库，进入 python 文件夹:

```
git clone [https://github.com/GoogleCloudPlatform/dataproc-templates.git](https://github.com/GoogleCloudPlatform/dataproc-templates.git)
cd dataproc-templates/python
```

**3。**执行 Hive to BigQuery 模板:

**a.** 导出工作提交所需的变量:

```
export GCP_PROJECT=<project_id> *# your Google Cloud project*
export REGION=<region> *# your region ex: us-central1*
export JARS="gs://spark-lib/bigquery/spark-bigquery-latest_2.12.jar" *# .jar dependencies*
export SUBNET=<subnet> *# optional if you are using default*
# export GCS_STAGING_LOCATION=<gcs-staging-bucket-folder> *# already done at step 1*
```

**b.** 运行 Dataproc 模板 shell 脚本，该脚本将读取上述变量，创建一个 Python 包，并将作业提交给 Dataproc Serverless。

**注意**:确保“python”可执行文件在您的路径中

```
./bin/start.sh \
- properties=spark.hadoop.hive.metastore.uris=thrift://<hostname-or-ip>:9083 \
- - template=HIVETOBIGQUERY \
- hive.bigquery.input.database="<database>" \
- hive.bigquery.input.table="<table>" \
- hive.bigquery.output.dataset="<dataset>" \
- hive.bigquery.output.table="<table>" \
- hive.bigquery.output.mode="<append|overwrite|ignore|errorifexists>" \
- hive.bigquery.temp.bucket.name="<temp-bq-bucket-name>"
```

您可能已经注意到，在提交模板时需要填写一些参数。这些参数在[模板文件](https://github.com/GoogleCloudPlatform/dataproc-templates/tree/main/python/dataproc_templates/hive)中有描述。

成功了！作业将会运行，您可以从 [Dataproc 无服务器 UI](https://console.cloud.google.com/dataproc/batches) 对其进行监控。

# 笔记

*   要定制作业，您可以从模板(之前克隆的)中更改 [PySpark 代码](https://github.com/GoogleCloudPlatform/dataproc-templates/blob/main/python/dataproc_templates/hive/hive_to_bigquery.py)。
*   为了快速拥有一个 Hive 环境来测试模板执行，您可以用 Apache Hive 创建一个 [Dataproc 集群，并连接到它。](https://cloud.google.com/architecture/using-apache-hive-on-cloud-dataproc)
*   假设 Apache Hive 的身份验证不是通过 Kerberos 进行的。否则，您必须使用必要的属性修改代码，以便使用 Kerberos 进行身份验证。
*   BigQuery 表(输出)不需要预先创建，但是数据集需要。
*   您可能已经注意到我们需要两个临时存储桶，一个用于 Dataproc Serverless 中的作业提交依赖项，另一个用作将数据加载到 big query(hive . big query . temp . bucket . name)的临时位置。
*   如果出现错误“当‘internal _ IP _ only’设置为‘true’时，子网‘default’不支持 Dataproc 集群所需的私有 Google 访问”,请确保查看并遵循前提条件部分的建议。在子网上启用私人 Google 访问“默认”或将“internal_ip_only”设置为“false”。
*   HiveToBigQuery 模板将从指定的表中读取所有分区，但是如果您想要处理来自某个特定分区的数据，更改代码很简单。

## 也是开源项目 [Dataproc 模板](https://github.com/GoogleCloudPlatform/dataproc-templates)的贡献者！