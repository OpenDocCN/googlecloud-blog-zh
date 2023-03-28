# 如何将 BigQuery 表(或数据集)备份到 Google 云存储中并从中恢复

> 原文：<https://medium.com/google-cloud/how-to-backup-a-bigquery-table-or-dataset-to-google-cloud-storage-and-restore-from-it-6ef7eb322c6d?source=collection_archive---------0----------------------->

## 将对 bq 命令行实用程序的调用串在一起

***注:此文已过时。BigQuery 现在支持*** [***表格快照***](https://cloud.google.com/bigquery/docs/table-snapshots-intro) ***。***

BigQuery 是完全托管的，负责管理存储中的冗余备份。它还支持“时间旅行”，即查询一天前的快照的能力。因此，如果 ETL 作业出错，而您想要恢复到昨天的数据，您只需执行以下操作:

```
CREATE OR REPLACE TABLE dataset.table_restored
AS 
SELECT *
FROM dataset.table
FOR SYSTEM TIME AS OF 
  TIMESTAMP_ADD(CURRENT_TIMESTAMP(), INTERVAL -1 DAY)
```

然而，时间旅行被限制为 7 天。有情况(回放，法规遵从性等。)当您可能需要将表恢复到 30 天前或 1 年前的状态时。

![](img/f98f5cad89fb9624b7931093589a2334.png)

## 用于备份和恢复的 Python 脚本

为了方便起见，我为备份和恢复 BigQuery 表和数据集编写了一对 Python 程序[和](https://github.com/GoogleCloudPlatform/bigquery-oreilly-book/tree/master/blogs/bigquery_backup)。从这个 GitHub 存储库中获取它们:[https://GitHub . com/Google cloud platform/big query-oreilly-book/tree/master/blogs/big query _ backup](https://github.com/GoogleCloudPlatform/bigquery-oreilly-book/tree/master/blogs/bigquery_backup)

以下是脚本的使用方法:

将表备份到 GCS

```
./bq_backup.py --input dataset.tablename --output gs://BUCKET/backup
```

该脚本以 AVRO 格式将 schema.json、tabledef.json 和提取的数据保存到 GCS。

您还可以备份数据集中的所有表:

```
./bq_backup.py --input dataset --output gs://BUCKET/backup
```

通过指定目标数据集逐个还原表

```
./bq_restore.py --input gs://BUCKET/backup/fromdataset/fromtable --output destdataset
```

## Python 脚本如何工作

这些脚本使用 BigQuery 命令行实用程序 bq 来组合一个备份和恢复解决方案。

[bq_backup.py](https://github.com/GoogleCloudPlatform/bigquery-oreilly-book/blob/master/blogs/bigquery_backup/bigquery_backup.py) 调用以下命令:

```
bq show --schema dataset.table. # schema.json
bq --format=json show dataset.table.  # tbldef.json
bq extract --destination_format=AVRO \
           dataset.table gs://.../data_*.avro # AVRO files
```

它将 JSON 文件和 AVRO 文件一起保存到谷歌云存储中。

[bq_restore.py](https://github.com/GoogleCloudPlatform/bigquery-oreilly-book/blob/master/blogs/bigquery_backup/bigquery_restore.py) 使用表定义来查找特性，例如表是时间分区的、范围分区的还是聚集的，然后调用命令:

```
bq load --source_format=AVRO \
    --time_partitioning_expiration ... \
    --time_partitioning_field ... \
    --time_partitioning_type ... \
    --clustering_fields ... \
    --schema ... \
    todataset.table_name \
    gs://.../data_*.avro
```

对于视图，脚本存储和恢复视图定义。视图定义是表定义 JSON 的一部分，要恢复它，脚本只需调用:

```
bq mk --view query --nouse_legacy_sql todataset.table_name
```

尽情享受吧！

## 1.需要备份到谷歌云存储吗？

在我发表这篇文章后，安托万·卡斯在推特上回应道:

他完全正确。本文假设您希望在云存储上进行文件形式的备份。这可能是因为您的合规性审计员希望将数据导出到某个特定位置，也可能是因为您希望备份可由其他工具处理。

如果不需要文件形式的备份，一种更简单的备份 BigQuery 表的方法是使用 *bq cp* 创建/恢复备份:

```
# backup
date=...
bq mk dataset_${date}
bq cp dataset.table dataset_${date}.table# restore
bq cp dataset_20200301.table dataset_restore.table
```

## 2.备份文件被压缩了吗？

是啊！数据以 Avro 格式存储，Avro 格式采用压缩。两个 JSON 文件(表定义和模式)没有被压缩，但是它们相对较小。

例如，我备份了一个有 4 亿行的 BigQuery 表，在 BigQuery 中占用了 11.1 GB。当在 GCS 上将它存储为 Avro 时，同一个表被保存为 47 个文件，每个文件都是 174.2 MB，也就是 8.2 GB。这两个 JSON 文件总共占用了 1.3 MB，基本上是四舍五入。这是有意义的，因为 BigQuery 存储针对交互式查询进行了优化，而 Avro 格式则没有。

## 3.你真的能在 BigQuery 和 Avro 之间来回吗？

BigQuery 可以将大多数原始类型和嵌套重复的字段导出到 Avro 中。有关 Avro 代表权的全部细节，请参见[文件](https://cloud.google.com/bigquery/docs/exporting-data#avro_export_details)。备份工具指定了 use_avro_logical_types，因此日期和时间将分别存储为日期和时间毫秒。

也就是说，您应该验证备份/恢复在您的表上工作。