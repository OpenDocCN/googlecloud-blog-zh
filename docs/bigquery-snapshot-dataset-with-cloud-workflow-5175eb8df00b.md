# BigQuery:使用云工作流的快照数据集

> 原文：<https://medium.com/google-cloud/bigquery-snapshot-dataset-with-cloud-workflow-5175eb8df00b?source=collection_archive---------0----------------------->

![](img/1a6fe2b996d213ea5b47b99b434b51d8.png)

数据是任何公司的**金矿**，而**数据仓库是存储和查询这些数据的最佳场所**。在**谷歌云上，BigQuery** 是最受欢迎的工具:**无服务器、可扩展、按需付费**；它可以在几秒钟内处理**亿字节的数据**。

BigQuery 特性提供了 **DML 语句:数据操作语言**，用于插入、更新或删除数据。因为你的黄金资源是珍贵的，你想确定 DML 不会破坏当前的数据值

> 你必须在 BigQuery 中备份你的数据

# 表快照功能

[表格快照功能](https://cloud.google.com/bigquery/docs/table-snapshots-intro)正是为此而设计的。您可以现在对您的数据执行**快照，或者在过去 7 天内的任何毫秒执行[时间旅行功能](https://cloud.google.com/bigquery/docs/time-travel)最大持续时间)。**

该功能会对您的数据进行快照，**只会跟踪更新或删除字段**的变化。那是卑鄙的:

*   如果您拍摄的数据**从未改变，则快照不会产生任何成本**。
*   如果您更改了所有数据，您将为快照表的等效存储付费

****原理与 Git 储存库*** *和提交更改历史相同。**

# *表快照限制*

*有一些[存在限制](https://cloud.google.com/bigquery/docs/table-snapshots-intro#limitations)，但它们并没有真正阻塞。最令人沮丧的是范围:*

> *您只能快照一个表！*

*您的**表属于一组同构的表**(原始摄取数据、数据集市、暂存转换等)，这些表被分组到一个数据集中。*

*因此，单独的表快照没有意义，但是必须备份整个数据集。*

# *云工作流解决方案*

*我毫不怀疑 BigQuery 团队意识到了现实世界的局限性，这个特性总有一天会发布的。
但是**在那之前**，云工作流可以帮助我们满足这一需求。*

## *建筑*

*这个基本示例将一个**源数据集** *(要拍摄快照的数据集)*作为参数，并且可选地将**目标数据集** *(存储快照的位置)*。然后，步骤如下*

*   ***检查目标数据集是否存在**。如果没有，使用与源数据集相同的名称创建它，并在结尾使用`_backup`
    *对于该示例，我将到期时间设置为 7 天，以便在该持续时间后自动删除快照。**
*   ***列出源数据集中的表格***
*   ***遍历这些表，并为目标数据集中的每个表创建一个快照**。
    *[*为了提高性能，使用了*](https://cloud.google.com/workflows/docs/reference/syntax/parallel-steps) `[*Parallel*](https://cloud.google.com/workflows/docs/reference/syntax/parallel-steps)` [*云工作流的特性*](https://cloud.google.com/workflows/docs/reference/syntax/parallel-steps)**
*   ***如果列表结果中有一个`nextPageToken`，**跳转到列表步骤，新的** `**pageToken**` **值设置**。***

## ***代码***

***下面是一个代码示例。***

```
***main:
  params: [param]
  steps:
    - assignStep:
        assign:
          - sourceDataset: ${param.source_dataset}
          - targetDataset: ${default(map.get(param, "target_dataset"),sourceDataset + "_backup")}
          - projectId: ${sys.get_env("GOOGLE_CLOUD_PROJECT_ID")}
          - location: "US"
          - maxResult: 100
          - pageToken: ""
    - check-target-dataset:
        try:
          call: googleapis.bigquery.v2.datasets.get
          args:
            projectId: ${projectId}
            datasetId: ${targetDataset}
        except:
          as: e
          steps:
            - known_errors:
                switch:
                  - condition: ${e.code == 404} *#Create dataset* steps:
                      - create-target-dataset:
                          call: googleapis.bigquery.v2.datasets.insert
                          args:
                            projectId: ${projectId}
                            body:
                               location: ${location}
                               datasetReference:
                                 datasetId: ${targetDataset}
                                 projectId: ${projectId}
                               defaultTableExpirationMs: 604800000 *#Snapshot expire after 7 days* next: list-tables
            - unhandled_exception:
                raise: ${e}
    - list-tables:
        call: googleapis.bigquery.v2.tables.list
        args:
          datasetId: ${sourceDataset}
          projectId: ${projectId}
          maxResults: ${maxResult}
          pageToken: ${pageToken}
        result: listResult
    - perform-snapshots:
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
                        copy:
                          destinationTable:
                            datasetId: ${targetDataset}
                            projectId: ${projectId}
                            tableId: ${table.tableReference.tableId}
                          operationType: "SNAPSHOT"
                          sourceTables:
                            - projectId: ${projectId}
                              datasetId: ${sourceDataset}
                              tableId: ${table.tableReference.tableId}
                          writeDisposition: "WRITE_EMPTY"
    - check-iterate-pages:
        switch:
          - condition: ${default(map.get(listResult, "nextPageToken"),"") != ""}
            steps:
              - loop-over-pages:
                  assign:
                    - pageToken: ${listResult.nextPageToken}
                  next: list-tables***
```

**使用以下命令部署您的工作流**

```
**gcloud workflows deploy <workflowName> \
 --source=<workflowFileName> \
 --service-account=<runtimeServiceAccountEmail>**
```

***运行时服务帐户* ***必须在目标项目上拥有 BigQuery 管理员角色*** *才能创建目标数据集和快照；只有* ***成为源数据集上的数据查看器*** *。***

**并运行它**

```
**gcloud workflows run <workflowName> \
  --data='{"source_dataset":"<YourDataset>"}'**
```

***您可以使用云调度程序定期调用该工作流***

# **保护您最珍贵的资产**

**这个工作流程非常简单明了。还有**限制**。例如:**

*   ****您不能连续运行工作流程两次**，因为快照将会存在，并且您将会遇到*已经存在*的错误。
    *您可以在目标数据集中添加日期，例如***
*   ****源项目和目标项目**相同。
    *你应该希望有一个快照项目为例。***
*   ****目标数据集区域是硬编码的**(并且必须是与源数据集相同的**)
    *您可以获取源数据集，获取该区域，并在与源数据集相同的区域中创建目标数据集。*****

**无论如何，这是一个保护您公司中最珍贵的东西的基本示例:**您的数据资产****