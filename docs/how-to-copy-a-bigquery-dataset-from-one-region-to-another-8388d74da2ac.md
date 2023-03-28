# 如何将 BigQuery 数据集从一个区域复制到另一个区域

> 原文：<https://medium.com/google-cloud/how-to-copy-a-bigquery-dataset-from-one-region-to-another-8388d74da2ac?source=collection_archive---------0----------------------->

## 使用 BigQuery 传输服务

复制 BigQuery 数据集和表的常用方法是使用 bq cp:

```
bq cp source_project:source_dataset.source_table \
      dest_dataset.dest_table
```

遗憾的是，copy 命令不支持跨区域复制，只支持同一区域内的复制。

![](img/e3f9950248014809101e60aada1a14c8.png)

要进行跨区域复制，可以使用 BigQuery 传输服务。

首先，创建目标数据集:

```
bq mk --location eu ch10eu
```

然后，创建一个数据源为 cross_region_copy 的转移配置:

```
bq mk --transfer_config **--data_source=cross_region_copy** \
   --params='{"source_dataset_id": "iowa_liquor_sales", "source_project_id": "bigquery-public-data"}' \
   --target_dataset=ch10eu --display_name=liquor \
   --schedule_end_time="$(date -v **+1H** -u +%Y-%m-%dT%H:%M:%SZ)"
```

因为数据传输服务是为了例行的复制，这将每 24 小时重复一次。在我的示例中，我将结束时间设置为从现在起 1 小时，这样传输只发生一次。

*请* [*明星本期*](https://issuetracker.google.com/issues/152689624) *如您愿意* ***bq cp*** *支持跨区副本。*