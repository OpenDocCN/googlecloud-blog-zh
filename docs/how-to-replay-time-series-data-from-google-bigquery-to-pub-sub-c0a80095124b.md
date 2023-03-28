# 如何从 Google BigQuery 向 Pub/Sub 重放时间序列数据

> 原文：<https://medium.com/google-cloud/how-to-replay-time-series-data-from-google-bigquery-to-pub-sub-c0a80095124b?source=collection_archive---------0----------------------->

这是一篇教程文章，解释了如何将 BigQuery 表中的时间序列数据重放到发布/订阅主题中。有几个您可能需要它的用例:

*   回溯测试。
*   演示/可视化。
*   集成测试。

用于在不同服务之间移动数据的 GCP 服务是数据流。虽然 Google 提供了很多数据流模板，但是没有一个可以将数据从 BigQuery 转移到 Pub/Sub。

这就是为什么我们开发了自己的工具来解决这个任务:[https://github.com/blockchain-etl/bigquery-to-pubsub](https://github.com/blockchain-etl/bigquery-to-pubsub)。它可以用来重放任何带有[时间戳](https://cloud.google.com/bigquery/docs/reference/standard-sql/data-types#timestamp-type)字段的 BigQuery 表。这是一个 Python 程序，它从一个分区的 BigQuery 表中顺序地提取数据块，并及时地将这些行作为 JSON 消息发布到一个 Pub/Sub 主题。下面是一个突出显示基本选项的示例:

```
> python run_command.py \
**--bigquery-table** bigquery-public-data.crypto_ethereum.transactions \
**--timestamp-field** block_timestamp \
**--start-timestamp** 2019-10-23T00:00:00 \
**--end-timestamp** 2019-10-23T01:00:00 \
--batch-size-in-seconds 1800 \
**--replay-rate** 0.1 \
**--pubsub-topic** projects/${project}/topics/bigquery-to-pubsub-test0 \
```

下面，您将在 BigQuery 中找到从[以太坊数据集中重放以太坊交易的详细说明。](https://console.cloud.google.com/marketplace/details/ethereum/crypto-ethereum-blockchain?filter=solution-type:dataset&filter=category:finance)

1.  创建具有以下角色的服务帐户:

*   BigQuery 管理员
*   存储管理员
*   发布/订阅发布者

2.为服务帐户创建一个密钥文件，并作为`credentials_file.json`下载。

3.创建一个名为`bigquery-to-pubsub-test0`的发布/订阅主题:

```
> gcloud pubsub topics create bigquery-to-pubsub-test0
```

4.创建临时 GCS 存储桶和临时 BigQuery 数据集:

```
> git clone [https://github.com/blockchain-etl/bigquery-to-pubsub.git](https://github.com/blockchain-etl/bigquery-to-pubsub.git)
> cd bigquery-to-pubsub
> bash create_temp_resources.sh
```

5.为以太坊交易运行重放:

```
> docker build -t bigquery-to-pubsub:latest -f Dockerfile .
> project=$(gcloud config get-value project 2> /dev/null)
> temp_resource_name=$(./get_temp_resource_name.sh)
> echo "Replaying Ethereum transactions"
> docker run \
-v /path_to_credentials_file/:/bigquery-to-pubsub/ --env GOOGLE_APPLICATION_CREDENTIALS=/bigquery-to-pubsub/credentials_file.json \
    bigquery-to-pubsub:latest \
--bigquery-table bigquery-public-data.crypto_ethereum.transactions \
--timestamp-field block_timestamp \
--start-timestamp 2019-10-23T00:00:00 \
--end-timestamp 2019-10-23T01:00:00 \
--batch-size-in-seconds 1800 \
--replay-rate 0.1 \
--pubsub-topic projects/${project}/topics/bigquery-to-pubsub-test0 \
--temp-bigquery-dataset ${temp_resource_name} \
--temp-bucket ${temp_resource_name}
```

## 在 Google Kubernetes 引擎中运行

对于长时间运行的重放，在云中启动这个过程很方便。Google Kubernetes 引擎是一个很好的选择，因为它提供了现成的 Stackdriver 日志记录和健康监控。

以下是使用[舵](https://helm.sh/)在 [GKE](https://cloud.google.com/kubernetes-engine) 中生成重放作业的说明:

1.  克隆 GitHub 存储库:

```
> git clone [https://github.com/blockchain-etl/bigquery-to-pubsub.git](https://github.com/blockchain-etl/bigquery-to-pubsub.git)
> cd bigquery-to-pubsub/helm
```

2.创建并初始化 GKE 集群、发布/订阅主题、临时存储桶和大查询数据集:

```
> bash scripts/setup.sh
```

3.复制和编辑示例值:

```
> cp values-sample.yaml values-dev.yaml
> vim values-dev.yaml
```

4.对于`tempBigqueryDataset`和`tempBucket`使用由`bash scripts/get_temp_resource_name.sh`打印的数值

5.安装图表:

```
> helm install bigquery-to-pubsub/ -name bigquery-to-pubsub-0 -values values-dev.yaml
```

6.检查`kubectl get pods`的输出。当状态为“已完成”时，作业完成。使用`kubectl logs <POD> -f`查看进度。

7.清理资源:

```
> bash scripts/cleanup.sh
```

接下来是一篇展示如何使用数据流和这个工具进行回溯测试的文章。