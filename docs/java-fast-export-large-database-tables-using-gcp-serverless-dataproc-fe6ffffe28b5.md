# [Java]快速导出大型数据库表——使用 GCP 无服务器 Dataproc

> 原文：<https://medium.com/google-cloud/java-fast-export-large-database-tables-using-gcp-serverless-dataproc-fe6ffffe28b5?source=collection_archive---------3----------------------->

## 将大型表从任何 JDBC (MySQL、PostgreSQL、MSSQL)导入到 BigQuery

![](img/d3f8dbf55fd5fc77050ec4617d611801.png)

Dataproc Serverless 是 Google Cloud Dataproc 平台的一个很好的补充。它允许用户运行 Spark 工作负载，而无需配置或管理集群。Dataproc Serverless 只是在幕后管理所有需要的基础设施。

Dataproc 模板为我们提供了这些工作负载的常见用例，而不需要我们自己开发。这些模板也让我们快速定制和运行它们。

## **简介**

如果您需要快速导入/导出 100 GBs-TBs 的大型表，请采用多线程并行导入/导出数据的方法，并使用可靠、成熟、开源和加固的机制。这篇文章可能对你有所帮助。

让我们使用 JDBCToBigQuery 模板以快速、高效和多线程的方式导出表。

## 要求

使用任何预装 JDK8+、Maven3+、Git 和 [gcloud CLI](https://cloud.google.com/sdk/docs/install) 的机器。
或者使用预装了这些工具的云壳。

# 简单用法

这种方法不是多线程的，所以它适用于小于 1Gb 的表。

1.  [建议]克隆您的活动实例，或创建读取副本。出于一致性目的，建议暂停对源数据库的写入。
2.  确保可以从 VPC 网络访问您的数据库。如果使用公共数据库，请确保启用云 NAT。请参考[本](https://cloud.google.com/dataproc-serverless/docs/concepts/network)了解更多信息。
3.  为 jar 文件创建一个 GCS 存储桶和暂存位置。为各自的源数据库下载 [JDBC 驱动 jar](https://github.com/GoogleCloudPlatform/dataproc-templates/blob/main/python/dataproc_templates/jdbc/README.md) ，以及带有 Spark 的 [BigQuery 连接器。将这些 jar 文件上传到 GCS Bucket 中。](https://cloud.google.com/dataproc/docs/tutorials/bigquery-connector-spark-example)
4.  克隆 Dataproc 模板 git repo:

```
git clone [https://github.com/GoogleCloudPlatform/dataproc-templates.git](https://github.com/GoogleCloudPlatform/dataproc-templates.git)
cd dataproc-templates/java
```

5.获取身份验证凭据:

```
gcloud auth application-default login
```

6.执行模板，有关更多详细信息，请参考 [JDBCToBigQuery](https://github.com/GoogleCloudPlatform/dataproc-templates/blob/main/java/src/main/java/com/google/cloud/dataproc/templates/jdbc/README.md) 文档。替换环境值以匹配您的情况(gcp 项目、区域、jdbc url、jar 路径等。)

```
export GCP_PROJECT=my-gcp-proj \
export REGION=us-central1  \
export SUBNET=projects/my-gcp-proj/regions/us-central1/subnetworks/default   \
export GCS_STAGING_LOCATION=gs://my-gcp-proj/mysql-export/staging \
export JARS="gs://my-gcp-proj/mysql-export/mysql-connector-java-8.0.17.jar,gs://my-gcp-proj/bigquery-jar/spark-bigquery-with-dependencies_2.12-0.23.2.jar"

bin/start.sh \
-- --template JDBCTOBIGQUERY \
--templateProperty jdbctobq.bigquery.location=bigquery-destination \
--templateProperty jdbctobq.jdbc.url=com.mysql.cj.jdbc.Driver \
--templateProperty jdbctobq.jdbc.driver.class.name=<jdbc driver class name> \
--templateProperty jdbctobq.sql="SELECT * FROM MyDB.employee" \
--templateProperty jdbctobq.write.mode=Overwrite \
--templateProperty jdbctobq.temp.gcs.bucket=temp-bucket-name
```

**注意**:如果尚未启用，它会要求您启用 Dataproc API。

# 高级用法(多线程导出/导入)

假设您在 mysql 数据库中有一个 Employee 表模式，如下所示:

```
CREATE TABLE `employee` (
  `id` bigint(20) unsigned NOT NULL,
  `name` varchar(100) NOT NULL,
  `email` varchar(100) NOT NULL,
  `current_salary` int unsigned DEFAULT NULL,
  `account_id` bigint(20) unsigned NOT NULL,
  `department` varchar(100) DEFAULT NULL,
  `created_at` datetime NOT NULL,
  `updated_at` datetime NOT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin;
```

假设最大雇员 id 为 1 亿(用于 upperBound 参数)。

按照上一节所述执行步骤 1-4。
通过指定分区属性来更改步骤 6。

执行 spark 作业和分区参数，示例如下:

```
export GCP_PROJECT=my-gcp-proj \
export REGION=us-central1  \
export SUBNET=projects/my-gcp-proj/regions/us-central1/subnetworks/default   \
export GCS_STAGING_LOCATION=gs://my-gcp-proj/mysql-export/staging \
export JARS="gs://my-gcp-proj/mysql-export/mysql-connector-java-8.0.17.jar,gs://my-gcp-proj/bigquery-jar/spark-bigquery-with-dependencies_2.12-0.23.2.jar"

bin/start.sh \
-- --template JDBCTOBIGQUERY \
--templateProperty jdbctobq.bigquery.location=bigquery-destination \
--templateProperty jdbctobq.jdbc.url=com.mysql.cj.jdbc.Driver \
--templateProperty jdbctobq.jdbc.driver.class.name=<jdbc driver class name> \
--templateProperty jdbctobq.sql="SELECT * FROM MyDB.employee" \
--templateProperty jdbctobq.write.mode=Overwrite \
--templateProperty jdbctobq.temp.gcs.bucket=temp-bucket-name \
**--templateProperty jdbctobq.sql.partitionColumn=id \
--templateProperty jdbctobq.sql.lowerBound=0 \
--templateProperty jdbctobq.sql.upperBound=100000000 \
--templateProperty jdbctobq.sql.numPartitions=400** 
```

# 另一个目标

1.  **另一个数据库**
    Spark JDBC 原生支持以下数据库 MySQL / MariaDB、Postgresql、DB2、Oracle。使用 [**GCSToJDBC**](https://github.com/GoogleCloudPlatform/dataproc-templates/blob/main/java/src/main/java/com/google/cloud/dataproc/templates/gcs/README.md#3-gcs-to-jdbc) 模板( [blogpost](/google-cloud/importing-data-from-gcs-to-databases-via-jdbc-using-dataproc-serverless-7ed75eab93ba) )您可以将数据摄取到其中任何一个模板中。
2.  从 Python 环境中运行 [JDBCToBigQuery](/@sjlva/python-fast-export-large-database-tables-using-gcp-serverless-dataproc-bfe77a132485) 。

# 参考

*   [Dataproc 无服务](https://cloud.google.com/dataproc-serverless/docs/overview)
*   [Dataproc 模板库](https://github.com/GoogleCloudPlatform/dataproc-templates)