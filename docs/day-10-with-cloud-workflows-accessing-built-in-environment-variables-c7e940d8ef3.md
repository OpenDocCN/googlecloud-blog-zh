# 云工作流第 10 天:访问内置环境变量

> 原文：<https://medium.com/google-cloud/day-10-with-cloud-workflows-accessing-built-in-environment-variables-c7e940d8ef3?source=collection_archive---------2----------------------->

[Google Cloud Workflows](https://cloud.google.com/workflows) 提供了一些内置的环境变量，您可以从工作流执行中访问这些变量。

目前有 5 个[环境变量](https://cloud.google.com/workflows/docs/reference/environment-variables)被定义:

*   GOOGLE_CLOUD_PROJECT_NUMBER:工作流项目的编号。
*   GOOGLE_CLOUD_PROJECT_ID:工作流项目的标识符。
*   GOOGLE_CLOUD_LOCATION:工作流的位置。
*   GOOGLE_CLOUD_WORKFLOW_ID:工作流的标识符。
*   GOOGLE _ CLOUD _ WORKFLOW _ REVISION _ ID:工作流的版本标识符。

让我们看看如何从我们的工作流定义中访问它们:

```
$ gcloud beta workflows deploy w09-new-workflow-from-cli \
    --source=w09-hello-from-gcloud.yaml \
    --service-account=workflow-sa@workflows-days.iam.gserviceaccount.com
```

我们使用内置的 sys.get_env()函数来访问这些变量。我们将在后面的章节中再次讨论各种现有的内置函数。

然后，当您执行此工作流时，您将得到如下输出:

```
“workflows-days 783331365595 europe-west4 w10-builtin-env-vars 000001–3af”
```

我希望这个列表中添加一个变量，那就是当前的执行 ID。当查看日志时，这对于识别特定的执行、推断潜在的失败或用于审计目的可能是有用的。

【http://glaforge.appspot.com】最初发表于[](http://glaforge.appspot.com/article/day-10-with-cloud-workflows-accessing-built-in-environment-variables)**。**