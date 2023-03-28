# 使用 BigQuery 导出获得单个 CSV 文件

> 原文：<https://medium.com/google-cloud/get-a-single-one-csv-file-with-bigquery-export-956d2a147886?source=collection_archive---------0----------------------->

![](img/8bac0c386c08031d949f7b63ddaba6c0.png)

**分布式计算**是大规模处理**大数据**的关键。所有数据处理系统都使用**虚拟机集群** : Hadoop、Dataflow，当然还有 BigQuery。
几十或几百个进程可以**同时运行**和**执行并行操作**以加速进程。

[BigQuery 导出语句](https://cloud.google.com/bigquery/docs/reference/standard-sql/other-statements#export_data_statement)是一个很好的例子:使用一条 SQL 语句，您可以**产生数百个槽** *(即 VM 的片)*来从 BigQuery 导出数据。但是因为**导出是由数百个插槽并行**执行的，所以**导出会生成数百个文件**。

我已经描述了[如何利用 BigQuery 来处理、合并、清理 CSV 文件](/google-cloud/merge-clean-transform-your-csvs-easily-with-bigquery-3a73c0c26d57)以及作为输出的几个 CSV 文件**的权衡**；
或[如何从 BigQuery 导出数据并加载到云 SQL](/google-cloud/replicate-data-from-bigquery-to-cloud-sql-2b23a08c52b1) 中，通过 BigQuery 对所有导出的 CSV 文件进行**迭代。[我的 GitHub repositor](https://github.com/guillaumeblaquiere/workflow-bq-to-cloudsql/issues/1) y 上有一个关于**伟大优化提议的问题:****

> 与其把很多小文件导入到 Cloud SQL 中，为什么不把所有文件合并，在 Cloud SQL 中只导入一个？

> 如何在一个 CSV 文件中从 BigQuery 导出数据？

让我们解决这个挑战

# 云存储合成功能

云存储**存储任何类型的二进制对象** (Blob)。当 BigQuery 执行导出时，**云存储用于存储那些导出的文件**。

云存储对于存储 blob 来说**非常高效(也很实惠)，但是它的**处理能力是有限的**:**

*   可以用[转码功能](https://cloud.google.com/storage/docs/transcoding)提供纯格式的 gzip 文件
*   可以对对象实施[生命周期](https://cloud.google.com/storage/docs/lifecycle)
*   [能在一个](https://cloud.google.com/storage/docs/composing-objects)中合成几个文件吗

*大致就这些！*

最新的要点,**并不广为人知，因为它不能通过控制台**获得。它允许你将 ***(追加)*多达 32 个文件组成一个单一的**。

但是这个特性被**限制为每个 API 调用**32 个文件，当你有数百个文件时，你需要**循环那个命令**。

> 如何自动循环合成上百个文件？

# 云工作流解决方案

[云工作流](https://cloud.google.com/workflows/docs)允许**通过 YAML 文件声明一系列 API 调用**，对流程进行控制([迭代](https://cloud.google.com/workflows/docs/reference/syntax/iteration)、[条件](https://cloud.google.com/workflows/docs/reference/syntax/conditions)、[重试](https://cloud.google.com/workflows/docs/reference/syntax/retrying)、…)。

因此一个**工作流是完美的**通过 BigQuery 浏览所有导出的文件并**将它们附加到一个单独的** CSV 文件中。

让我们来定义工作流！

# BigQuery 导出调用

第一步是用导出语句查询 big query**。使用的格式是 CSV，因为通过附加文件** *(无头文件)*很容易合并 CSV。

[BigQuery 作业查询连接器](https://cloud.google.com/workflows/docs/reference/googleapis/bigquery/v2/jobs/query)用于执行该查询。
*运行工作流的服务帐户必须对要查询的数据拥有“BigQuery 数据查看者”角色，并对项目拥有“BigQuery 作业用户”角色，才能运行查询作业。*

```
- export-query:
    call: googleapis.bigquery.v2.jobs.query
    args:
      projectId: ${projectid}
      body:
        query: ${"EXPORT DATA OPTIONS( uri='gs://" + bucket + "/" + prefix + "*.csv', format='CSV', overwrite=true,header=false) AS " + query}
        useLegacySql: false
```

*`*header=false*`*很重要，避免在每个文件中添加头，然后在追加文件时发出。**

# *云存储管理*

*云存储是该工作流程的核心服务。使用了许多不同的 API 和功能。*

**在该部分中，运行工作流的服务帐户必须拥有存储对象 Admin(查看者+创建者)，才能执行以下操作。**

## *输出文件*

*作为输出，我**想要一个包含所有导出数据的新文件**。我不想更改 BigQuery 生成的文件，以保持 BigQuery 过程的完整性，并在出现问题时重新运行 compose 操作。*

*所以，我们的想法是**创建一个初始文件，所有其他的 CSV 将被附加到这个文件中**。
*目前(2021 年 12 月)连接器有问题，团队正在处理。因此，我直接使用* [*云存储 JSON API*](https://cloud.google.com/storage/docs/json_api/v1/objects/insert)*

```
*- create-final-file:
    call: http.post
    args:
      url: ${"https://storage.googleapis.com/upload/storage/v1/b/" + bucket + "/o?name=" + finalFileName + "&uploadType=media"}
      auth:
        type: OAuth2
      body:*
```

**该文件可以为空(my case，* `*body:*` *为空)，但是如果您的用例中需要一个标题，您可以手动添加一个标题。**

## *列出文件*

*云存储的另一个用途是文件列表**获取所有导出的文件**。好消息是 BigQuery 导出语句**需要一个云存储文件前缀**来存储导出的文件。
云存储**只能过滤文件前缀**。*

*我们只需重用同一个就行了！*

*为此，我们使用[云存储对象列表连接器](https://cloud.google.com/workflows/docs/reference/googleapis/storage/v1/objects/list)*

```
*- list-files:
    call: googleapis.storage.v1.objects.list
    args:
      bucket: ${bucket}
      pageToken: ${pagetoken}
      prefix: ${prefix}
      maxResults: 62
    result: listResult*
```

**我将在优化部分*讨论 `*maxResults*` *和* `*pageToken*` *参数**

## *遍历文件*

*现在我们已经列出了所有文件，我们**需要迭代它们**，当我们**累积了 32 个文件**(合成限制)，或者**到达文件列表的末尾**时，我们可以**运行云存储合成操作。***

*迭代基于云存储列表对象返回的文件列表。我们**在一个列表**中追加文件名，当**列表达到 32 个元素时，一个组合操作被触发**。*

```
*- init-iter:
    assign:
      - finalFileFormatted:
          name: ${finalFileName}
      - fileList:
          - ${finalFileFormatted}
- process-files:
    for:
      value: file
      in: ${listResult.items}
      steps:
        - concat-file:
            assign:
              - fileFormatted:
                  name: ${file.name}
              - fileList: ${list.concat(fileList, fileFormatted)}
        - test-concat:
            switch:
              - condition: ${len(fileList) == 32} 
                steps:
                  - compose-files:
                      call: compose_file
                      args:
                        fileList: ${fileList}
                        projectid: ${projectid}
                        bucket: ${bucket}
                        finalFileName: ${finalFileName}
                      next: init-for-iter
                  - init-for-iter:
                      assign:
                        - fileList:
                            - ${finalFileFormatted}
- finish-compose: *# Process the latest files in the fileList buffer* switch:
      - condition: ${len(fileList) > 1} *# If there is more than the finalFileName in the list* steps:
          - last-compose-files:
              call: compose_file
              args:
                fileList: ${fileList}
                projectid: ${projectid}
                bucket: ${bucket}
                finalFileName: ${finalFileName}*
```

**列表不是空的，第一个文件是期望的输出文件，所有其他文件都附加到它后面。**

## *撰写最终文件*

*最后也是最重要的**，作曲操作**。这里，我们是使用连接器的**并将所有文件**附加到`destinationObject`中。*

```
*- compose:
    call: googleapis.storage.v1.objects.compose
    args:
      destinationBucket: ${bucket}
      destinationObject: ${text.replace_all(finalFileName,"/","%2F")}
      userProject: ${projectid}
      body:
        sourceObjects: ${fileList}*
```

*如你所见，`destinationObject`是“特殊的”。我们必须转换它以避免不兼容的字符。这里的`/`被替换为`%2F`，URL 编码的等价物。**您可以使用类似的替换来执行带有空格字符的**或`<>`。该团队正在开发新的编码功能来自动完成这项工作。*

# *部署并运行它*

*您可以在我的 [GitHub 库](https://github.com/guillaumeblaquiere/workflow-bq-export-to-one-csv)中找到完整的代码，并按照`README.md`文件中的说明来部署和运行工作流。*

## *问题和优化*

*在工作流 YAML 定义中，我包含了一些优化和问题修复。*

*首先，**当 BigQuery 导出结束和 compose 操作之间的过程太快**时，**云存储上找不到文件** *(我不知道为什么)*。我添加了一个 1 秒的睡眠计时器来解决这个延迟问题*

```
*- waitForGCS: *# fix latency issue with Cloud Storage* call: sys.sleep
    args:
      seconds: 1*
```

*然后为了**优化(即最小化)对 Compose API** 的调用数量，我以 31 的倍数列出了对象。
为什么 31？
因为**限制是 32 个文件组成**，我们必须**考虑最终输出文件**。因此**只有 31 个新文件要与最终输出文件**合成。*

```
*- list-files:
    call: googleapis.storage.v1.objects.list
    args:
      bucket: ${bucket}
      pageToken: ${pagetoken}
      prefix: ${prefix}
      maxResults: 62*
```

*但是为什么是 62？不是 93，124 或者更多？**来了一个工作流程的限制**。变量大小有一个[限制。今天**是 64kb，很快就是 256Kb** ，但是你不能在一个变量中存储太多的结果而不破坏你的执行。](https://cloud.google.com/workflows/quotas#resource_limits)*

*最新的问题是**防止过多的迭代深度**。如果递归调用一个子工作流，可以得到[一个执行](https://cloud.google.com/workflows/docs/reference/syntax/error-types) `[RecursionError](https://cloud.google.com/workflows/docs/reference/syntax/error-types)`。
解决方案是**在子工作流本身之外执行检查和迭代**并**再次调用它**直到递归结束*(仍要处理的页面)**

```
*- composefiles:
    call: list_and_compose_file
    args:
      pagetoken: ${listResult.nextPageToken}
      bucket: ${bucket}
      prefix: ${prefix}
      projectid: ${projectid}
      finalFileName: ${finalFileName}
    result: listResult
- missing-files: *# Non recursive loop, to prevent depth errors* switch:
      - condition: ${"nextPageToken" in listResult}
        next: composefiles*
```

# *组合、优化、创新*

*云工作流**是一个很好的工具，可以轻松编排 API 调用**。您可以组合产品，而无需部署代码、编写应用程序或设置复杂的架构。你**可以解决以前需要花费更多努力才能实现的挑战**。*

*该产品尚未完善，一些开发者功能仍在路线图中，但**已经提供了优雅的解决方案，** it **发展非常快**并且**已经普遍可用(GA)** 。试试吧！*