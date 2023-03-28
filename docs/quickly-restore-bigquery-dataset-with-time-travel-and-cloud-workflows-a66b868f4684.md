# 使用时间旅行和云工作流快速恢复 BigQuery 数据集

> 原文：<https://medium.com/google-cloud/quickly-restore-bigquery-dataset-with-time-travel-and-cloud-workflows-a66b868f4684?source=collection_archive---------0----------------------->

![](img/16a85e40ad90c1506566b73647d7aac4.png)

数据对公司来说是**黄金**。数据湖和数据仓库年复一年地变得流行起来，以保持和利用这种价值。

> 更好的**数据质量和丰富度**，更高的**其价值和更强大或更准确的**是你可以用它实现的用例。

然而，数据管道**是 IT 资产**，正如其中任何一个一样，潜在的**错误、问题或错误配置**会**搞乱您的数据、降低质量或破坏一致性**。

在谷歌云上， [BigQuery](https://cloud.google.com/bigquery) 提供了一个强大的**功能，可以回到大灾难之前的**！

# BigQuery 时间旅行

BigQuery 是**谷歌云数据仓库旗舰**。它是无服务器的，你按使用付费，有大量的功能。
其中一个，不太为人所知的是[时间旅行](https://cloud.google.com/bigquery/docs/time-travel):你可以在过去 7 天的任何时间点访问你的**数据状态。**

要使用它，添加您想要在查询中查看的`system time`。

```
SELECT *
FROM `mydataset.mytable`
  FOR SYSTEM_TIME AS OF <DATE|TIMESTAMP>
```

## 在变更日期扩展你的知识

除了时间旅行特性，一个全新的**提供了了解 BigQuery 表的** [**变化历史**](https://cloud.google.com/bigquery/docs/change-history) 的功能。

您可以执行这种类型的请求

```
SELECT
* except (_CHANGE_TYPE,_CHANGE_TIMESTAMP),
_CHANGE_TYPE AS change_type,
_CHANGE_TIMESTAMP AS change_time
FROM
APPENDS(TABLE mydataset.mytable, TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY), NULL);
```

*这样，你就可以看到一行什么时候被更新了，* ***更好地了解你的问题的爆炸半径。***

# 还原 BigQuery 表

知道数据的过去值是很有趣的。但是现在想象一下

*   用户执行查询，使**改变**，甚至**删除**数据
*   一个夜间**进程被更新**并扰乱你所有的数据
*   周一，你**发现周五晚上流程的一个错误配置**

> 您希望**恢复某个时间点的数据**

然后，使用固定的管道重放您的工作负载，以尽可能保持高质量的数据。

为此，您可以在您表中执行查询来**获取引用状态并存储在另一个表中**

```
CREATE TABLE mydataset.mynewtable AS 
SELECT *
FROM `mydataset.mytable`
  FOR SYSTEM_TIME AS OF <DATE|TIMESTAMP>
```

那你可以

*   **删除当前表格的数据**
*   **将保存的数据**插入当前表格
*   **删除**临时新表

对于数据集的所有表，也是如此。 ***好无聊！！***

让我们实现自动化吧！

# 自动化数据集恢复

对于自动化来说，[**云工作流**](https://cloud.google.com/workflows) 是一个有用的服务，可以循环所有数据集的表并调用 BigQuery APIs。

因为，我们直接使用 API，我们可以**用[big query 作业 API](https://cloud.google.com/bigquery/docs/reference/rest/v2/jobs) 简化过程**。在查询[工作中插入](https://cloud.google.com/bigquery/docs/reference/rest/v2/jobs/insert):

*   **定义源表中的查询**与时间行程
*   **设置目标**表等于源表，
*   **将处置设置为** `**WRITE_TRUNCATE**`以在恢复前清除现有数据

像这样，用**一个单独的 API 调用**，我们就可以**直接恢复源表**中的数据！

让我们深入工作流代码。

## 分配变量

首先分配全局变量和参数变量

```
assign:
          - sourceDataset: ${param.source_dataset}
          - date: ${default(map.get(param,"date"),"TIMESTAMP_ADD(CURRENT_TIMESTAMP(), INTERVAL -1 DAY)")}
          - projectId: ${sys.get_env("GOOGLE_CLOUD_PROJECT_ID")}
          - location: "US"
          - maxResult: 100
          - pageToken: ""
```

*在那个例子中* `*date*` *是可选的。如果省略，将采用昨天的数据。* ***日期是 SQL 注入的(不要让脚本对任何人开放！！).*** *但这也意味着你可以设置一个 SQL 公式。*

## 列出数据集中的表

第一步是了解要恢复的表。大多数情况下，**数据集**中的整个表必须在某个时间点恢复，以**保持数据一致**。

使用了[大查询表列表连接器](https://cloud.google.com/workflows/docs/reference/googleapis/bigquery/v2/tables/list)

```
- list-tables:
        call: googleapis.bigquery.v2.tables.list
        args:
          datasetId: ${sourceDataset}
          projectId: ${projectId}
          maxResults: ${maxResult}
          pageToken: ${pageToken}
        result: listResult
```

## 并行运行查询作业

有了这个表列表，我们就可以遍历它并调用 [BigQuery 作业插入连接器](https://cloud.google.com/workflows/docs/reference/googleapis/bigquery/v2/jobs/insert)。
为了优化流程，我们使用云工作流的[并行特性](https://cloud.google.com/workflows/docs/reference/syntax/parallel-steps)

```
- perform-restore:
        parallel:
          for:
            value: table
            in: ${listResult.tables}
            steps:
              - snapshot:
                  call: googleapis.bigquery.v2.jobs.insert
                  args:
                    projectId: ${projectId}
                    body:
                      configuration:
                        query:
                          useLegacySql: false
                          destinationTable:
                            datasetId: ${sourceDataset}
                            projectId: ${projectId}
                            tableId: ${table.tableReference.tableId}
                          query: ${"select * from " + projectId + "." + sourceDataset + "." + table.tableReference.tableId + " FOR SYSTEM TIME AS OF " + date }
                          writeDisposition: "WRITE_TRUNCATE"
```

*注* `*date*` *参数注入在时间旅行部分。您还可以注意到查询中的表和目的地中的表是相同的*

## 遍历表的页面

如果数据集有许多表，则列表查询可以有几页。迭代它们

```
- check-iterate-pages:
        switch:
          - condition: ${default(map.get(listResult, "nextPageToken"),"") != ""}
            steps:
              - loop-over-pages:
                  assign:
                    - pageToken: ${listResult.nextPageToken}
                  next: list-tables- check-iterate-pages:
        switch:
          - condition: ${default(map.get(listResult, "nextPageToken"),"") != ""}
            steps:
              - loop-over-pages:
                  assign:
                    - pageToken: ${listResult.nextPageToken}
                  next: list-tables
```

这是全部代码

```
main:
  params: [param]
  steps:
    - assignStep:
        assign:
          - sourceDataset: ${param.source_dataset}
          - date: ${default(map.get(param,"date"),"TIMESTAMP_ADD(CURRENT_TIMESTAMP(), INTERVAL -1 DAY)")}
          - projectId: ${sys.get_env("GOOGLE_CLOUD_PROJECT_ID")}
          - location: "US"
          - maxResult: 100
          - pageToken: ""
    - list-tables:
        call: googleapis.bigquery.v2.tables.list
        args:
          datasetId: ${sourceDataset}
          projectId: ${projectId}
          maxResults: ${maxResult}
          pageToken: ${pageToken}
        result: listResult
    - perform-restore:
        parallel:
          for:
            value: table
            in: ${listResult.tables}
            steps:
              - snapshot:
                  call: googleapis.bigquery.v2.jobs.insert
                  args:
                    projectId: ${projectId}
                    body:
                      configuration:
                        query:
                          useLegacySql: false
                          destinationTable:
                            datasetId: ${sourceDataset}
                            projectId: ${projectId}
                            tableId: ${table.tableReference.tableId}
                          query: ${"select * from " + projectId + "." + sourceDataset + "." + table.tableReference.tableId + " FOR SYSTEM TIME AS OF " + date }
                          writeDisposition: "WRITE_TRUNCATE"
    - check-iterate-pages:
        switch:
          - condition: ${default(map.get(listResult, "nextPageToken"),"") != ""}
            steps:
              - loop-over-pages:
                  assign:
                    - pageToken: ${listResult.nextPageToken}
                  next: list-tables
```

使用以下命令部署您的工作流

```
gcloud workflows deploy <workflowName> \
 --source=<workflowFileName> \
 --service-account=<runtimeServiceAccountEmail>
```

*运行时服务帐户* ***必须拥有数据集上的 BigQuery admin 角色*** *。*

并运行它

```
gcloud workflows run <workflowName> \
  --data='{"source_dataset":"<YourDataset>"}'
```

你也可以提到`date`作为输入

```
gcloud workflows run <workflowName> \
  --data='{"source_dataset":"<YourDataset>", "date":"2022-09-15"}'
```

# 保持您的数据价值

您的**数据在您的数据仓库**中至关重要，而 **BigQuery 和云工作流**的结合可以帮助您完成这项任务。

我已经介绍过[如何给你的数据集](/google-cloud/bigquery-snapshot-dataset-with-cloud-workflow-5175eb8df00b)拍快照，以便能够超越 7 天的时间旅行。
这一次，在数据损坏的情况下，这是一个可能的**快速回滚**。

**问题发生**，错误存在**，想办法**保护核心价值**！**