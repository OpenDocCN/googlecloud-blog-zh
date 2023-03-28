# 使用 Dataproc 无服务器将 GC 迁移到 GC

> 原文：<https://medium.com/google-cloud/migrate-gcs-to-gcs-using-dataproc-serverless-3b7b0f6ad6b9?source=collection_archive---------3----------------------->

D

> 此外， [Dataproc 模板](https://github.com/GoogleCloudPlatform/dataproc-templates/blob/main/README.md)提供了预定义的作业，可以按原样使用，也可以根据您的需求进行定制。因此，总而言之，这减少了管理基础设施和构建 spark 代码的总时间。

![](img/50ddbcafc308b3c038e017459a5dbf01.png)

使用 Dataproc 的 GCStoGCS 迁移

# **目标**

这篇博文讲述了如何利用[**GCSToGCS**](https://github.com/GoogleCloudPlatform/dataproc-templates/blob/main/java/src/main/java/com/google/cloud/dataproc/templates/gcs/README.md#4-gcs-to-gcs)data proc 无服务器模板进行数据迁移。此外，这篇文章涵盖了两种主要编码语言(即 **Python** 和 **Java。**

## 设置您的 GCP 项目和基础设施

1.  登录到您的 GCP 项目并启用 Dataproc API，如果禁用的话
2.  如果你要使用 GCP 生成的“默认”VPC 网络，请确保该子网启用了私人谷歌访问。您仍然需要启用如下的私人访问

![](img/a46e23ea1c26661b8108c5579470ce13.png)

```
gcloud compute networks subnets update default --region=us-central1 --enable-private-ip-google-access
```

3.为 jar 文件创建一个 GCS 存储桶和暂存位置。

```
export GCS_STAGING_BUCKET=”my-gcs-staging-bucket”
gsutil mb gs://$GCS_STAGING_BUCKET
```

## 执行 Dataproc 模板的步骤

1.  克隆 Dataproc 模板库并导航到 Java 模板文件夹。

```
git clone https://github.com/GoogleCloudPlatform/dataproc-templates.git
cd dataproc-templates/java
```

要导航到 Python 模板文件夹，请执行以下命令:

```
git clone https://github.com/GoogleCloudPlatform/dataproc-templates.git
cd dataproc-templates/python
```

2.获取身份验证凭据(以提交作业)。

```
gcloud auth application-default login
```

3.通过导出提交所需的变量来配置 Dataproc 无服务器作业—

`GCP_PROJECT`:运行 Dataproc 无服务器的 GCP 项目 id。

`REGION`:运行 Dataproc 无服务器的区域。

`GCS_STAGING_LOCATION` : GCS 暂存区位置，Dataproc 将在此存储暂存资产(参见 GCP 项目设置部分中的步骤 3)。

4.**【Python 模板】**收集以下可选参数的值

*   `GCS.TO.GCS.INPUT.FORMAT` : GCS 输入文件格式(avro、parquet、csv、json 中的一种)
*   `GCS.TO.GCS.INPUT.LOCATION`:输入文件的 GCS 位置
*   `GCS.TO.GCS.OUTPUT.FORMAT` : GCS 输入文件格式(avro、parquet、csv、json 中的一种)
*   `GCS.TO.GCS.OUTPUT.LOCATION`:目标文件的 GCS 位置
*   `GCS.TO.GCS.OUTPUT.MODE`:输出写入模式(追加、覆盖、忽略、错误存在中的一种)(默认为追加)
*   `GCS.TO.GCS.TEMP.VIEW.NAME`:在源数据上创建 spark sql 视图的临时视图名。
*   `GCS.TO.GCS.SQL.QUERY:`:数据转换的 SQL 查询。

> ***注意:*** *当使用转换属性时，Spark 临时视图的名称和查询中视图的名称应该完全匹配，以避免“找不到表/视图”的错误。*

使用强制环境变量执行提供的`bin/start.sh`脚本，将作业提交给 dataproc 无服务器——下面是 Python 模板的一个示例执行命令:

```
./bin/start.sh \
-- --template=GCSTOGCS \ 
    --gcs.to.gcs.input.location="<gs://bucket/path>" \
    --gcs.to.gcs.input.format="<json|csv|parquet|avro>" \
    --gcs.to.gcs.output.location="<gs://bucket/path>"
    --gcs.to.gcs.output.format="<json|csv|parquet|avro>" \
    --gcs.to.gcs.output.mode="<append|overwrite|ignore|errorifexists>" \
    --gcs.to.gcs.temp.view.name="temp" \
    --gcs.to.gcs.sql.query="select *, 1 as col from temp" \
```

5.**【Java 模板】**收集以下可选参数的值

*   `GCS.TO.GCS.INPUT.FORMAT` : GCS 输入文件格式(avro、parquet、csv、json 中的一种)
*   `GCS.TO.GCS.INPUT.LOCATION`:输入文件的 GCS 位置
*   `GCS.TO.GCS.OUTPUT.FORMAT` : GCS 输入文件格式(avro、parquet、csv、json 中的一种)
*   `GCS.TO.GCS.OUTPUT.LOCATION`:目标文件的 GCS 位置
*   `GCS.TO.GCS.OUTPUT.MODE`:输出写入模式(追加、覆盖、忽略、错误存在中的一种)(默认为追加)
*   `GCS.TO.GCS.TEMP.TABLE`:在源数据上创建 spark sql 视图的临时表名。
*   `GCS.TO.GCS.TEMP.QUERY:`:数据转换的 SQL 查询。

> ***注意:*** *当使用转换属性时，Spark 临时表的名称和查询中的表的名称应该完全匹配，以避免“找不到表/视图”的错误。*

使用强制环境变量执行提供的`bin/start.sh`脚本，将作业提交给 dataproc 无服务器——下面是 Java 模板的一个示例执行命令:

```
bin/start.sh \
-- --template GCSTOJDBC \
--templateProperty project.id=my-gcp-project \
--templateProperty gcs.gcs.input.location=gs://my-gcp-project-input-bucket/filename.avro \
--templateProperty gcs.gcs.input.format=avro \
--templateProperty gcs.gcs.output.location=gs://my-gcp-project-output-bucket \
--templateProperty gcs.gcs.output.format=csv \
--templateProperty gcs.jdbc.output.saveMode=Overwrite 
--templateProperty gcs.gcs.temp.table=temp \
--templateProperty gcs.gcs.temp.query='select *, 1 as col from temp'
```

**提供火花特性**

如果您需要[指定 Dataproc 无服务器支持的 spark 属性](https://cloud.google.com/dataproc-serverless/docs/concepts/properties)，例如调整驱动程序、内核、执行器等的数量，您可以编辑 start.sh 文件中的 [OPT_PROPERTIES](https://github.com/GoogleCloudPlatform/dataproc-templates/blob/main/java/bin/start.sh#L50) 值。

6.监控 Spark 批处理作业

提交作业后，我们将能够在 [Dataproc Batches UI](https://console.cloud.google.com/dataproc/batches) 中看到。从那里，我们可以查看作业的指标和日志。

## 参考

*   [Dataproc 无服务](https://cloud.google.com/dataproc-serverless/docs/overview)
*   [Dataproc 模板库](https://github.com/GoogleCloudPlatform/dataproc-templates)

如有任何疑问/建议，请联系:dataproc-templates-support-external@googlegroups.com