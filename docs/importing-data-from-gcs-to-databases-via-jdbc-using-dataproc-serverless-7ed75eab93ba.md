# 使用 Dataproc 无服务器将数据从 GCS 导入数据库(通过 JDBC)

> 原文：<https://medium.com/google-cloud/importing-data-from-gcs-to-databases-via-jdbc-using-dataproc-serverless-7ed75eab93ba?source=collection_archive---------0----------------------->

![](img/e2034781b0068df463c9718f16355b47.png)

在运行 Spark 作业的同时管理服务器始终是一项挑战。对 Spark 作业使用完全托管的按需服务器是当今时代的需要。它帮助开发人员专注于核心应用程序逻辑，而不是花时间管理框架。Dataproc Serverless 就是 Google 云平台提供的一个这样的产品。

![](img/819cf1296dfc0581447091724c646c58.png)

大多数事务数据仍然驻留在关系数据库服务器上，可以通过使用 JDBC 驱动程序来连接。MySQL、Oracle 和 SQL Server 主要用于此目的。

![](img/252ab408451fd6d27e8504cc0ee7352f.png)

当今世界正在转向基于云的存储服务来存储数据。它引发了谷歌云存储桶的使用。

这篇文章是关于通过 Dataproc 无服务器将数据从 GCS 桶转移到 JDBC 数据库的。

# 主要优势

1.  使用 **Dataproc 无服务器**运行 Spark 批处理工作负载，无需管理 Spark 框架。批量大小也可以在该模板中配置。
2.  [**GCSToJDBC**](https://github.com/GoogleCloudPlatform/dataproc-templates/blob/main/java/src/main/java/com/google/cloud/dataproc/templates/gcs/README.md#3-gcs-to-jdbc) 模板是开源的，配置驱动的，随时可以使用。执行代码只需要 JDBC 和 GCS 证书。
3.  支持的文件格式有 Avro、Parquet 和 ORC。
4.  [**JDBCToGCS**](https://github.com/GoogleCloudPlatform/dataproc-templates/blob/main/java/src/main/java/com/google/cloud/dataproc/templates/jdbc/README.md#2-jdbc-to-gcs) 模板可以反过来使用，即通过 JDBC 将数据从数据库导出到 GCS 存储桶。

# **用途**

1.  如果你要使用“默认的”由 GCP 生成的 VPC 网络，请确保你已经启用了私有谷歌访问子网。您仍然需要启用如下的私人访问。

![](img/75fd723625ca3e21aa6f036e53c5fff0.png)

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

5.执行 GCSToJDBC 模板。
例如:

```
export GCP_PROJECT=my-gcp-project
export REGION=us-central1
export GCS_STAGING_LOCATION=gs://staging-bucket \
export JARS=gs://jar-location/mysql-connector-java-8.0.29.jarbin/start.sh \
-- --template GCSTOJDBC \
--templateProperty project.id=my-gcp-project \
--templateProperty gcs.jdbc.input.location=gs://inputBucket/empavro \
--templateProperty gcs.jdbc.input.format=avro \
--templateProperty gcs.jdbc.output.table=demo \
--templateProperty gcs.jdbc.output.saveMode=Overwrite \
--templateProperty gcs.jdbc.output.url='jdbc:mysql://12.345.678.9:3306/test?user=root&password=root' \
--templateProperty gcs.jdbc.output.driver='com.mysql.jdbc.Driver' \
--templateProperty gcs.jdbc.output.batchInsertSize=1000
```

**注意**:如果尚未启用，它会要求您启用 Dataproc Api。

# 设置每日增量数据导出

有时需要日/周/月末导出。最重要的是，您可能希望只提取增量变更，因为在每次迭代中导出整个数据可能会有些过头。

**注意** : Dataproc Serverless 不支持实时作业。因此，如果目标是将变更从 GCS 实时复制到 JDBC，那么您将需要考虑变更数据捕获(CDC)的替代方案。

1.  使用标准数据库的[提交时间戳](https://cloud.google.com/spanner/docs/commit-timestamp)特性，您可以跟踪插入/更新的行(删除不能被捕获)。
2.  重写提交给 spark 的 sql 查询，以捕获自上次执行以来的更改。例如:

```
jdbctogcs.sql='select trip_id, bikeid, duration_minutes from bikeshare where LastUpdateTime >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 1 DAY'
```

3.计划批处理作业。
GCP 原生提供云调度+云功能，可用于提交 spark 批量作业。或者自我管理的软件，如 linux cron tab，Jenkins 等。也可以使用。

# 设置附加火花属性

如果您需要[指定 Dataproc 无服务器支持的 spark 属性](https://cloud.google.com/dataproc-serverless/docs/concepts/properties)，如调整驱动程序、内核、执行器等的数量。

您可以编辑 start.sh 文件中的 [OPT_PROPERTIES](https://github.com/GoogleCloudPlatform/dataproc-templates/blob/main/java/bin/start.sh#L50) 值。

**参考文献**
[https://medium . com/Google-cloud/cloud-spanner-export-query-results-using-data proc-server less-6 F2 f 65 b 583 a4](/google-cloud/cloud-spanner-export-query-results-using-dataproc-serverless-6f2f65b583a4)
[https://cloud.google.com/pubsub/docs/overview](https://cloud.google.com/dataproc-serverless/docs/overview)
[https://github.com/GoogleCloudPlatform/dataproc-templates](https://github.com/GoogleCloudPlatform/dataproc-templates)