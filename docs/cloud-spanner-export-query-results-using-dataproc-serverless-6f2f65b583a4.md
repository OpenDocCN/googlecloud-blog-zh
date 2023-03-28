# 使用 Dataproc 无服务器导出查询结果

> 原文：<https://medium.com/google-cloud/cloud-spanner-export-query-results-using-dataproc-serverless-6f2f65b583a4?source=collection_archive---------1----------------------->

![](img/21e39f07bd450e9d8d995cccb4999fbd.png)

Cloud Spanner 是完全托管的关系数据库，规模不限，一致性强，可用性高达 99.999%。在这篇文章中，我们将探索使用 Dataproc Serverless 为扳手表或 SQL 查询导出数据。

# 主要优势

1.  Dataproc Serverless 是完全管理的、无服务器的和自动伸缩的。
2.  [**SpannerToGCS**](https://github.com/GoogleCloudPlatform/dataproc-templates/tree/main/java/src/main/java/com/google/cloud/dataproc/templates/databases#executing-spanner-to-gcs-template) 模板是开源的，完全可定制的，可以随时用于简单的工作。
3.  一旦作业结束，短暂火花资源就被释放。
4.  您可以以 **avro、parquet、csv 和 orc** 格式导出数据。

# 简单用法

1.  如果你要使用“默认的”由 GCP 生成的 VPC 网络，请确保你已经启用了私有谷歌访问子网。您仍然需要启用如下的私人访问。([详见此处](https://cloud.google.com/dataproc-serverless/docs/concepts/network))

![](img/3b1552393115c07b39e070abf1da4389.png)

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

5.执行 SpannerToGCS 模板。
例如:

```
export GCP_PROJECT=my-test-gcp-proj1
export REGION=us-central1
export GCS_STAGING_LOCATION=gs://my-test-gcp-proj1/stagingbin/start.sh \
-- --template SPANNERTOGCS \
--templateProperty project.id=my-test-gcp-proj1 \
--templateProperty spanner.gcs.input.spanner.id=spann \
--templateProperty spanner.gcs.input.database.id=testdb \
--templateProperty "spanner.gcs.input.table.id=(select trip_id, bikeid, duration_minutes from bikeshare)" \
--templateProperty spanner.gcs.output.gcs.path=gs://my-test-gcp-proj1/bikeshare/spark_export/ \
--templateProperty spanner.gcs.output.gcs.saveMode=Overwrite
```

**注意**:如果尚未启用，它会要求您启用 Dataproc Api。

# 设置每日增量数据导出

有时需要日/周/月末导出。最重要的是，您可能希望只提取增量变更，因为在每次迭代中导出整个数据可能会有些过头。

**注意**:无服务器 Dataproc 不支持实时作业。因此，如果目标是从 Spanner 到 GCS 实时复制变更，那么您将需要研究变更数据捕获(CDC)替代方案。

1.  使用 Cloud Spanner 的[提交时间戳](https://cloud.google.com/spanner/docs/commit-timestamp)特性，您可以跟踪插入/更新的行(删除不能被捕获)。
2.  重写提交给 spark 的 sql 查询，以捕获自上次执行以来的更改。例如:

```
"spanner.gcs.input.table.id=(select trip_id, bikeid, duration_minutes from bikeshare where LastUpdateTime >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 1 DAY);"
```

3.计划批处理作业。
GCP 原生提供云调度+云功能，可用于提交 spark 批量作业。或者，也可以使用自我管理软件，如 linux cron tab、jenkins 等。

# 设置附加火花属性

如果您需要[指定 Dataproc 无服务器支持的 spark 属性](https://cloud.google.com/dataproc-serverless/docs/concepts/properties),比如调整驱动程序、内核、执行器等的数量。

您可以编辑 start.sh 文件中的 [OPT_PROPERTIES](https://github.com/GoogleCloudPlatform/dataproc-templates/blob/main/java/bin/start.sh#L50) 值。

# 其他选择

1.  基于 GUI 的工具 [DBeaver](https://dbeaver.io/) 将 sql 查询结果导出为 csv 格式。
2.  [Cloud Spanner 联邦查询](https://cloud.google.com/bigquery/docs/cloud-spanner-federated-queries)使用 BigQuery 将数据导出为 Avro、CSV 和其他支持的格式。
3.  用于[扳手到文本](https://cloud.google.com/dataflow/docs/guides/templates/provided-batch#cloud-spanner-to-cloud-storage-text)导出为 csv 文件或[扳手到 Avro](https://cloud.google.com/spanner/docs/export#exporting_a_subset_of_tables) 的数据流模板