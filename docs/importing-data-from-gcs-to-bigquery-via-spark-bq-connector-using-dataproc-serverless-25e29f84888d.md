# 使用 Dataproc 无服务器将数据从 GCS 导入 Bigquery(通过 Spark BQ 连接器)

> 原文：<https://medium.com/google-cloud/importing-data-from-gcs-to-bigquery-via-spark-bq-connector-using-dataproc-serverless-25e29f84888d?source=collection_archive---------2----------------------->

![](img/0c64cbe368912a30b2179f0bd6aa1d02.png)

在无服务器处理时代，在专用集群上运行 Spark 作业会增加更多的处理开销，并占用开发人员宝贵的开发时间。将完全托管的随需应变服务器与 Spark jobs 结合使用，有助于开发人员专注于核心应用程序逻辑，在框架上花费很少或没有时间。谷歌的 Dataproc Serverless 就是谷歌云平台提供的这样一款产品。

将数据从异构数据源引入 Bigquery 时，最常见的接收模式是使用 GCS 作为着陆区，在将数据加载到 bigquery 表(ETL)或直接加载到 bigquery 表并进行转换(ELT)之前，可以从这里转换数据。

在本文中，我们将讨论无服务器 Dataproc 如何帮助将数据从 GCS 加载到 bigquery，用于 ETL 或 ELT 目的。

# 主要优势

1.  使用 **Dataproc 无服务器**运行 Spark 批处理工作负载，无需管理 Spark 框架。
2.  [**gcstobiqquery**](https://github.com/GoogleCloudPlatform/dataproc-templates/blob/main/java/src/main/java/com/google/cloud/dataproc/templates/gcs/GCStoBigquery.java)模板是开源的，配置驱动的，随时可以使用。执行代码只需要 GCS 凭证。
3.  支持的文件格式有 Avro、Parquet 和 CSV。
4.  [**BigqueryToGCS**](https://github.com/GoogleCloudPlatform/dataproc-templates/blob/main/java/src/main/java/com/google/cloud/dataproc/templates/bigquery/BigQueryToGCS.java) 模板可以反过来使用，即通过 JDBC 将数据从数据库导出到 GCS 存储桶。

# 使用

1.  如果你要使用“默认的”由 GCP 生成的 VPC 网络，请确保你已经启用了私有谷歌访问子网。您仍然需要启用如下的私人访问。

![](img/cb458f67d72301f15b6a6e6d42008504.png)

```
gcloud compute networks subnets update default --region=us-central1 --enable-private-ip-google-access
```

2.为 jar 文件创建一个 GCS 存储桶和暂存位置。

3.在预装了[各种工具](https://cloud.google.com/shell/docs/how-cloud-shell-works)的云壳中克隆 git repo。或者使用任何预装 JDK 8+，Maven 和 Git 的机器。

```
git clone [https://github.com/GoogleCloudPlatform/dataproc-templates.git](https://github.com/GoogleCloudPlatform/dataproc-templates.git)cd dataproc-templates/java
```

4.获取身份验证凭据(以提交作业)。

```
gcloud auth application-default login
```

5.执行 GCSToBigquery 模板。
例如:

```
export GCP_PROJECT=my-gcp-project
export REGION=us-central1
export GCS_STAGING_LOCATION=gs://staging-bucket \
export JARS=gs://jar-location/mysql-connector-java-8.0.29.jar
java/bin/start.sh \
-- --template GCSTOBIGQUERY \
--templateProperty project.id=<gcp-project-id> \
--templateProperty gcs.bigquery.input.location=<gcs path> \
--templateProperty gcs.bigquery.input.format=<csv|parquet|avro|orc> \
--templateProperty gcs.bigquery.output.dataset=<datasetId> \
--templateProperty gcs.bigquery.output.table=<tableName> \
--templateProperty gcs.bigquery.temp.bucket.name=<bigquery temp bucket name>
```

**注意**:如果尚未启用，它会要求您启用 Dataproc Api。

6.到目前为止，这个过程将把数据从您的 gcs 位置加载到 bigquery。如果您想要转换数据，模板提供了一种方法，一旦将源数据加载到数据集中，就可以对其执行自定义 sql。

```
--templateProperty gcs.bigquery.temp.table='temporary_view_name' 
--templateProperty gcs.bigquery.temp.query='select * from global_temp.temporary_view_name'
```

上述属性负责在将数据加载到 BigQuery 时应用一些 spark sql 转换。唯一需要记住的是，Spark 临时视图的名称和查询中的表名应该完全匹配。否则，将会出现如下错误:“找不到表或视图”