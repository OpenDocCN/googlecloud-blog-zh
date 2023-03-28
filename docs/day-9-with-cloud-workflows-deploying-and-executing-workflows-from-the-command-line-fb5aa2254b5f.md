# 云工作流第 9 天:从命令行部署和执行工作流

> 原文：<https://medium.com/google-cloud/day-9-with-cloud-workflows-deploying-and-executing-workflows-from-the-command-line-fb5aa2254b5f?source=collection_archive---------1----------------------->

到目前为止，在这个关于[云工作流](https://cloud.google.com/workflows)的系列中，我们只使用了谷歌云控制台 UI 来管理我们的工作流定义及其执行。但是也可以使用。让我们看看如何做到这一点！

如果您还没有现有的服务帐户，您应该按照这些[说明](https://cloud.google.com/workflows/docs/creating-updating-workflow#gcloud)创建一个。在本次演示中，我将使用我创建的 workflow-sa 服务帐户。

我们的工作流定义是一个简单的“hello world ”,就像我们为探索 Google Cloud 工作流而创建的一样:

```
- hello:
    return: Hello from gcloud!
```

为了部署这个工作流定义，我们将启动以下 gcloud 命令，指定我们工作流的名称，传递本地源定义和服务帐户:

```
$ gcloud beta workflows deploy w09-new-workflow-from-cli \
    --source=w09-hello-from-gcloud.yaml \
    --service-account=workflow-sa@workflows-days.iam.gserviceaccount.com
```

您还可以添加带有标签标志的标签，以及带有描述标志的描述，就像在 Google Cloud 控制台 UI 中一样。

如果您想要更新工作流定义，这也是要调用的同一个命令，传递您的定义文件的新版本。

是时候创建我们工作流的执行了！

```
$ gcloud beta workflows run w09-new-workflow-from-cli
```

您将看到类似如下的输出:

```
Waiting for execution [d4a3f4d4-db45-48dc-9c02-d25a05b0e0ed] to complete...done.
argument: 'null'
endTime: '2020-12-16T11:32:25.663937037Z'
name: projects/783331365595/locations/us-central1/workflows/w09-new-workflow-from-cli/executions/d4a3f4d4-db45-48dc-9c02-d25a05b0e0ed
result: '"Hello from gcloud!"'
startTime: '2020-12-16T11:32:25.526194298Z'
state: SUCCEEDED
workflowRevisionId: 000001-47f
```

我们的工作流非常简单，它会立即执行并完成，因此您会看到结果字符串(来自 gcloud 的 Hello！消息)，以及状态为成功。然而，工作流通常需要更长的时间来执行，包括许多步骤。如果工作流尚未完成，您将看到它的状态为“活动”,或者如果出现问题，可能会失败。

当工作流需要很长时间才能完成时，您可以使用以下命令从 shell 会话中检查上次执行的状态:

```
$ gcloud beta workflows executions describe-last
```

如果您想了解正在进行的工作流执行情况:

```
$ gcloud beta workflows executions list your-workflow-name
```

它会给你一个正在执行的操作 id 列表。然后，您可以通过以下方式检查特定的一个:

```
$ gcloud beta workflows executions describe the-operation-id
```

对执行还有其他操作，等待执行完成，甚至取消正在进行的、未完成的执行。

您可以在[文档](https://cloud.google.com/workflows/docs/executing-workflow)中了解更多关于工作流执行的信息。在接下来的几集中，我们还将了解如何从客户端库和云工作流 REST API 创建工作流执行。

*原载于*[*http://glaforge.appspot.com*](http://glaforge.appspot.com/article/day-9-with-cloud-workflows-deploying-and-executing-workflows-from-the-command-line)*。*