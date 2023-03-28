# 使用 Dataproc 无服务器将数据从 Hive 加载到 Bigquery(通过 Spark BQ 连接器)

> 原文：<https://medium.com/google-cloud/loading-data-from-hive-to-bigquery-via-spark-bq-connector-using-dataproc-serverless-3a870d0ff64f?source=collection_archive---------5----------------------->

![](img/c66ef4b5c5f939d66a1fa2c0f5a3dff0.png)

在无服务器处理时代，在专用集群上运行 Spark 作业会增加更多的处理开销，并占用开发人员宝贵的开发时间。将完全托管的随需应变服务器与 Spark jobs 结合使用，有助于开发人员专注于核心应用程序逻辑，而在框架上花费很少或没有时间。谷歌的 Dataproc Serverless 就是谷歌云平台提供的这样一款产品。

在本文中，我们将讨论无服务器 Dataproc 如何通过 sql 将数据从 Hive 表加载到 bigquery，用于 ETL 或 ELT 目的。

# 主要优势

1.  使用 **Dataproc 无服务器**运行 Spark 批处理工作负载，无需管理 Spark 框架。
2.  [**hivetobiqquery**](https://github.com/GoogleCloudPlatform/dataproc-templates/blob/main/java/src/main/java/com/google/cloud/dataproc/templates/hive/HiveToBigQuery.java)模板是开源的，配置驱动的，随时可以使用。
3.  支持的加载模式有追加、覆盖、错误存在和忽略。

# 使用

1.  如果你要使用“默认的”由 GCP 生成的 VPC 网络，请确保你已经启用了私有谷歌访问子网。您仍然需要启用如下的私人访问。

![](img/516511504ac158645daf746635ffef55.png)

```
gcloud compute networks subnets update default --region=us-central1 --enable-private-ip-google-access
```

2.为 jar 文件创建一个 GCS 存储桶和暂存位置。

3.在预装了[各种工具](https://cloud.google.com/shell/docs/how-cloud-shell-works)的云壳中克隆 git repo。或者使用任何预装 JDK 8+，Maven 和 Git 的机器。

```
git clone https://github.com/GoogleCloudPlatform/dataproc-templates.gitcd dataproc-templates/java
```

4.获取身份验证凭据(以提交作业)。

```
gcloud auth application-default login
```

5.执行 HiveToBigquery 模板。
例如:

```
GCP_PROJECT=<gcp-project-id> \
REGION=<region>  \
SUBNET=<subnet>   \
GCS_STAGING_LOCATION=<gcs-staging-bucket-folder> \
HISTORY_SERVER_CLUSTER=<history-server> \
bin/start.sh \
--properties=spark.hadoop.hive.metastore.uris=thrift://<hostname-or-ip>:9083 \
-- --template HIVETOBIGQUERY \
--templateProperty hivetobq.bigquery.location=<project.dataset.tableß∂> \
--templateProperty hivetobq.sql=<hive_sql> \
--templateProperty hivetobq.write.mode=<Append|Overwrite|ErrorIfExists|Ignore> \ 
--templateProperty hivetobq.temp.gcs.bucket=<gcs_bucket_path>
```

双引号中的 Hive SQL 查询。例子，

```
--templateProperty  hivetobq.sql="select * from dbname.tablename"
```

**注意**:如果尚未启用，它会要求您启用 Dataproc Api。

6.到目前为止，这个过程将把数据从 Hive 表/查询加载到 bigquery。如果您想要转换数据，模板提供了一种方法，一旦将源数据加载到数据集中，就可以对其执行自定义 sql。

```
--templateProperty hivetobq.temp.table='temporary_view_name' 
--templateProperty hivetobq.temp.query='select * from global_temp.temporary_view_name'
```

上述属性负责在将数据加载到 BigQuery 时应用一些 spark sql 转换。唯一需要记住的是，Spark 临时视图的名称和查询中的表名应该完全匹配。否则，将会出现如下错误:“找不到表或视图”