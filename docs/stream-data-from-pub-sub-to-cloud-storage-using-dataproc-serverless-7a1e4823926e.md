# 使用 Dataproc 无服务器将数据从发布/订阅流式传输到云存储

> 原文：<https://medium.com/google-cloud/stream-data-from-pub-sub-to-cloud-storage-using-dataproc-serverless-7a1e4823926e?source=collection_archive---------0----------------------->

![](img/e6da9d1ffb6506f52120e2a7dc4db5de.png)

Google Cloud Pub/Sub 用于流分析和数据集成管道，以接收和分发数据。在这篇文章中，我们将探索如何使用 **Dataproc 无服务器**将数据从**发布/订阅**主题传输到**谷歌云存储**。

# 主要优势

1.  使用 **Dataproc Serverless** 运行 Spark batch 工作负载，无需配置和管理您自己的集群。
2.  [**PubSubToGCS**](https://github.com/GoogleCloudPlatform/dataproc-templates/blob/main/java/src/main/java/com/google/cloud/dataproc/templates/pubsub/README.md#2-pubsub-to-gcs) 模板是开源的，完全可定制，可用于简单的工作。
3.  您可以将数据从发布/订阅以 **AVRO 和 JSON** 格式传输到 GCS。

# 基本用法

1.  如果你要使用“默认的”由 GCP 生成的 VPC 网络，请确保你已经启用了私有谷歌访问子网。您仍然需要启用如下的私人访问。

![](img/a46e23ea1c26661b8108c5579470ce13.png)

`gcloud compute networks subnets update default — region=us-central1 — enable-private-ip-google-access`

2.为 jar 文件创建一个 GCS 存储桶和一个暂存文件夹。

3.创建一个带有模式的发布/订阅主题(对 AVRO 数据流使用二进制序列化)。还要为此发布/订阅主题创建订阅。

4.在预装了[各种工具](https://cloud.google.com/shell/docs/how-cloud-shell-works)的云壳中克隆 git repo。或者，你可以使用任何预装了 JDK 8+，Maven 和 Git 的机器。

`git clone [https://github.com/GoogleCloudPlatform/dataproc-templates.git](https://github.com/GoogleCloudPlatform/dataproc-templates.git) cd dataproc-templates/java`

5.获取认证凭证(以提交 Dataproc 作业)。

`gcloud auth application-default login`

6.对 GCS Dataproc 无服务器模板执行发布/订阅:

```
export GCP_PROJECT=my-gcp-project-id
export REGION=project-region
export GCS_BUCKET=gcs-bucket-name
export GCS_STAGING_LOCATION=gs://gcs-bucket-name/tempexport SUBNET=projects/$GCP_PROJECT/regions/$REGION/subnetworks/NAME
export SUBSCRIPTION=pubsub-subscription-name
# DATA_FORMAT can be either JSON or AVRO
export DATA_FORMAT=JSON
export BATCH_SIZE=50bin/start.sh \
-- \
--template PUBSUBTOGCS \
--templateProperty pubsubtogcs.input.project.id=$GCP_PROJECT \
--templateProperty pubsubtogcs.input.subscription=$SUBSCRIPTION \
--templateProperty pubsubtogcs.gcs.output.project.id=$GCP_PROJECT \
--templateProperty pubsubtogcs.gcs.bucket.name=$GCS_BUCKET \
--templateProperty pubsubtogcs.gcs.output.data.format=$DATA_FORMAT \
--templateProperty pubsubtogcs.batch.size=$BATCH_SIZE \
```

**注意**:如果尚未启用，它会要求您启用 Dataproc Api。

## 设置附加火花属性

如果您需要[指定 Dataproc 无服务器支持的 spark 属性](https://cloud.google.com/dataproc-serverless/docs/concepts/properties)，比如调整驱动程序、内核、执行器等的数量。您可以编辑 start.sh 文件中的 [OPT_PROPERTIES](https://github.com/GoogleCloudPlatform/dataproc-templates/blob/main/java/bin/start.sh#L50) 值。

**参考文献**
[https://medium . com/Google-cloud/cloud-spanner-export-query-results-using-data proc-server less-6 F2 f 65 b 583 a4](/google-cloud/cloud-spanner-export-query-results-using-dataproc-serverless-6f2f65b583a4)
[https://cloud.google.com/pubsub/docs/overview](https://cloud.google.com/pubsub/docs/overview)
[https://github.com/GoogleCloudPlatform/dataproc-templates](https://github.com/GoogleCloudPlatform/dataproc-templates)