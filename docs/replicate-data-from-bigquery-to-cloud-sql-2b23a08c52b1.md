# 使用云工作流将数据从 BigQuery 复制到云 SQL

> 原文：<https://medium.com/google-cloud/replicate-data-from-bigquery-to-cloud-sql-2b23a08c52b1?source=collection_archive---------1----------------------->

![](img/18d6c5ca8e946c2faf0713cfa7157e68.png)

随着流行和云的力量，一些人认为独角兽是存在的:**一个可以统治它们的数据库！**
然而，技术和物理都无法实现这个梦想。因此，每个云提供商都提出了**许多数据存储选项，每个选项都适合非常具体的用例。**

# 大查询问题

分析数据库，如 BigQuery，在几秒钟内处理数 Pb 的数据是非常高效的。聚合、复杂查询、数十亿行的连接为公司创造了新的洞察力和新的价值。

不好的一面是延迟。为了处理大量数据，在开始查询之前，需要提供一组 CPU/内存对(在 BigQuery 中称为“slots”)。而且就算很快，也就 1 秒左右。*即使数据量很低，或者查询很简单，过程也是一样的。*

这不是分析的问题，但不是实时的，如网站或 API 服务。

> 需要像云 SQL 这样的低延迟数据库。

# 云 SQL 选项

云 SQL 是托管数据库服务，可以**运行 MySQL、PostgreSQL 和 SQL server** 。这些数据库引擎可以在几毫秒内回答简单的查询，非常适合网站和 API 服务器。

然而，**云 SQL 数据库不能横向扩展**(并行添加额外的服务器来处理数据)，而**只能纵向扩展**(增加一台服务器上的 CPU 和内存的数量)。*读取副本(并行的附加服务器)也可以只在一个服务器(副本)上运行查询。*

因此，**性能低下**无法有效处理**大量数据**(限制为 10tb)

# 以低延迟提供强大的洞察力

**企业需要强大的洞察力**，能够计算和深入挖掘数 Pb 的数据，并能够以低延迟浏览结果**。**

**最理想的是结合两个世界，但是独角兽并不存在！**

> **解决方案是在 BigQuery 中执行计算，并将结果导出到 Cloud SQL 以降低延迟。**

## **数据重复问题**

**数据复制可能很可怕，或者被视为反模式。没那么明显。**

*   ****数据重复增加成本！**
    是的，是真的。然而，数据**存储成本比数据处理成本约低 10 倍**，复制数据以优化处理**可以节省资金**！**
*   ****重复数据失控。**
    这可能是真的。在这种情况下，我们**需要定义“金源”。**这里是 BigQuery，**云 SQL 数据只在这里只读**。不允许更新。*双向更新可能是一场管理噩梦；在这里，对于报告数据，我们用“主/复制”模式来避免这种担心。***

# **将 BigQuery 数据加载到云 SQL**

**[BigQuery 允许**导出 CSV 文件**](https://cloud.google.com/bigquery/docs/exporting-data#bq) 中的数据，并将文件存储在云存储中。
[云 SQL 允许**从云存储中导入 CSV 文件**](https://cloud.google.com/sql/docs/mysql/import-export/import-export-csv#import_data_from_a_csv_file) 。**

**原则上，这个过程似乎是显而易见的。**然而，事情没那么简单！****

*   **BigQuery 在几个文件中导出大量数据**。****
*   **云存储**一次只能导入一个文件**。不支持并发导入**

> **为了对这个过程进行排序，需要一个编排。云工作流非常适合这一点！**

**我们来写这个管道吧！**

## **导出 BigQuery 数据**

**[BigQuery 可以**导出存储在**表](https://cloud.google.com/bigquery/docs/exporting-data#bq)中的数据。它是免费的，但是你**不能选择你想要的数据**或者**格式化/转换它们**。它是以 CSV 格式将 BigQuery 存储复制并粘贴到云存储中。**

**这并不理想，因为我们必须**正确格式化要插入到云 SQL** 中的数据。希望存在另一个选项:[出口声明](https://cloud.google.com/bigquery/docs/reference/standard-sql/other-statements#export_data_statement)。**

**该语句接受 export 的选项和参数中的查询。**查询的结果存储在云存储中的一个或多个文件**中。
一个简单的**查询执行导出**！还有一个[工作流连接器用于运行查询！](https://cloud.google.com/workflows/docs/reference/googleapis/bigquery/v2/jobs/insert)**

**让我们从这一步开始我们的工作流程。**

```
- export-query:
    call: googleapis.bigquery.v2.jobs.query
    args:
      projectId: ${projectid}
      body:
        query: ${"EXPORT DATA OPTIONS( uri='gs://" + bucket + "/" + prefix + "*.csv', format='CSV', overwrite=true,header=false) AS " + query}
        useLegacySql: false
```

***因为你必须* ***插入一个作业并读取数据*** *，运行工作流的服务账号必须* ***有权限访问数据*** *(至少是* `*bigquery.dataViewer*` *)并且* ***能够在项目中创建一个作业*** *(是* `*bigquery.jobUser*` *)。要在云存储中写入导出文件，需要角色* `*storage.objectAdmin*` *。***

## **在云 SQL 中导入数据**

**现在，我们必须将数据导入到云 SQL 中。可以对云 SQL 管理 API 执行一个 **API 调用。
但是，这次**没有连接器**，我们不得不**直接调用 API。******

```
- callImport:
    call: http.post
    args:
      url: ${"https://sqladmin.googleapis.com/v1/projects/" + projectid + "/instances/" + instance + "/import"}
      auth:
        type: OAuth2
      body:
        importContext:
          uri: ${file}
          database: ${databaseschema}
          fileType: CSV
          csvImportOptions:
            table: ${importtable}
    result: operation
```

***为调用，可以* ***注*** `***OAuth2***` ***auth 引数*** *。需要* ***允许工作流给 API 调用*** *添加认证头。工作流服务帐户需要是云 SQL admin 才能执行导入。***

**因为:**

*   ****操作是异步的****
*   **没有用于异步操作的连接器**
*   **如果要导入多个文件，一次只能导入**一个文件****

**我们必须:**

*   **通过**定期检查当前导入操作的状态**获得当前导入的结束。**
*   **当工作完成后，**继续过程**:迭代其他文件或退出。**

**为此，我们**可以在导入调用的`result`中获得状态**。这个变量被命名为`operation`。
*如果状态不是* `*DONE*` *，我们必须等待并再次检查，否则我们可以退出。***

```
- chekoperation:
    switch:
      - condition: ${operation.body.status != "DONE"}
        next: wait
    next: completed
- completed:
    return: "done"
- wait:
    call: sys.sleep
    args:
      seconds: 1
    next: getoperation
- getoperation:
    call: http.get
    args:
      url: ${operation.body.selfLink}
      auth:
        type: OAuth2
    result: operation
    next: chekoperation
```

***API 很好的提供了一个* `*selflink*` *来获取运行状态。由于它，检查查询是最容易的。***

## **浏览云存储中的导出文件**

**我们有这个过程的两个方面:出口和进口。**我们必须绑定它们**，而**云存储就是建立关系的地方**。但是……**这种联系并不容易！****

**事实上，如果您仔细查看导入定义，您可以**一次只导入一个文件**。而 **BigQuery 导出可以创建一组文件**。**

**因此，我们必须在导出期间**迭代 BigQuery** 创建的文件，并为每个文件调用云 SQL 流程中的**导入步骤(或[子工作流](https://cloud.google.com/workflows/docs/reference/syntax/subworkflows)****

```
- list-files:
    call: googleapis.storage.v1.objects.list
    args:
      bucket: ${bucket}
      pageToken: ${pagetoken}
      prefix: ${prefix}
    result: listResult
- process-files:
    for:
      value: file
      in: ${listResult.items}
      steps:
        - wait-import:
            call: load_file
            args:
              projectid: ${projectid}
              instance: ${instance}
              databaseschema: ${databaseschema}
              importtable: ${importtable}
              file: ${"gs://" + bucket + "/" + file.name}1
```

**另外，如果你有大量的文件，**云存储列表 API 答案可以包含几页**。我们必须遍历文件和页面。**

**正如你在 [my GitHub repository](https://github.com/guillaumeblaquiere/workflow-bq-to-cloudsql) 中看到的，子工作流`list_file_to_import` **不是递归的，在浏览下一页时不会调用自己**。将**整体结果** `**listResult**` **发送回调用者**。**

**事实上，云**工作流在子工作流调用的深度**上有一个限制，如果达到这个限制，你可以[获得一个](https://cloud.google.com/workflows/docs/troubleshooting#execution_errors) `[RecursionError](https://cloud.google.com/workflows/docs/troubleshooting#execution_errors)`。
所以，**诀窍是退出子工作流**，用所需的信息给**让调用者选择用不同的参数再次调用同一个子工作流**，在那种情况下是`nextPageToken`。**

***你可以在 GitHub 资源库* *中找到* [*完整的工作流代码。您需要自定义分配步骤来设置您的值，或者更新此第一步以从工作流参数中动态获取值。
如果您有问题要自动处理，您可能还需要添加一些错误管理检查*](https://github.com/guillaumeblaquiere/workflow-bq-to-cloudsql/blob/main/import.yaml)**

# **生产您的同步**

**随着工作流的构建和部署，您必须在需要时执行它。可以是一个时间表，用 [**云调度器**](https://cloud.google.com/workflows/docs/schedule-workflow) ，也可以是一个事件。**

**现在，你必须**编写一个云函数(或云运行/应用引擎)来捕捉事件并运行一个工作流**执行。不过，很快，你就可以用[**event arc**](https://cloud.google.com/eventarc/docs)**开箱**做**了。****

**数据分析和低延迟数据库**是互补的**，为了利用两者的力量，一个**安全和协调的工作流是达到下一个水平的正确途径**。**