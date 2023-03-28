# 从函数中调用工作流！

> 原文：<https://medium.com/google-cloud/call-a-workflow-from-a-function-51fc7fc8e1ff?source=collection_archive---------1----------------------->

![](img/31984f6fc53eb69027c3fc4db7ee8121.png)

工作流+功能

云工作流允许您编排和自动化 Google 云和基于 HTTP 的 API 服务。

您可能会问，如何将这些工作流与您现有的云功能相集成？—很好的问题。

在这篇文章中，你将学习如何从云函数中调用工作流！🚀

# 部署云工作流

首先，您的项目中需要一个工作流(如果您还没有的话)。用以下内容创建一个名为`myFirstWorkflow.yaml`的新文件:

myFirstWorkflow.yaml

阅读上面的 YAML，工作流执行以下操作:

1.  从公共云功能获取当前日期/时间。
2.  用查询参数`dayOfTheWeek`调用 Wikipedia (opensearch) API。
3.  从维基百科返回最热门的结果。

让我们用`gcloud`部署工作流:

```
gcloud workflows deploy myFirstWorkflow \
--source=myFirstWorkflow.yaml
```

如果您愿意，可以通过以下方式测试工作流程:

```
gcloud workflows run myFirstWorkflow
```

使用该命令，您将执行工作流并等待结果。
响应将包括如下详细信息:

```
result: '["Tuesday","Tuesday Weld","Tuesday Night Music Club","Tuesday (ILoveMakonnen
  song)","Tuesday Group","Tuesdays with Morrie","Tuesday Knight","Tuesday (Burak Yeter
  song)","Tuesday Morning Quarterback","Tuesday Vargas"]'

state: SUCCEEDED
```

不错！✔️

# 创建云函数

所以现在我们不从`gcloud`运行工作流，而是从云函数调用工作流。您可以想象这对于创建发票或运行图像处理管道非常有用。

云函数使得调用工作流变得相当容易，因为我们不需要编写任何授权代码。让我们使用函数框架创建一个轻量级节点函数。

## 创建一个`package.json`文件:

package.json

这个文件描述了我们的节点程序，它使用了[功能框架](https://github.com/GoogleCloudPlatform/functions-framework-nodejs)和[云工作流 API 客户端库](https://github.com/googleapis/nodejs-workflows)。

## 创建一个`index.js`文件:

对于您的函数，创建以下`index.js`文件:

索引. js

下面是代码的简要说明:

*   `exports.runWorkflow`是你的云函数，传递给你的 Express `(req, res)`参数。我们接受 POST 请求，调用`callWorkflowsAPI`函数并返回结果。
*   有一个云工作流 API 客户端的小设置
*   `callWorkflowsAPI`函数创建我们工作流的执行，然后使用指数补偿等待工作流完成执行。
*   如果得到响应，我们立即返回，否则我们将超时并返回`{ "success": false }`。

如果您愿意，可以删除日志记录和指数补偿部分，以使代码更短。

> 要在本地主机上测试，运行`npm i`和`npm start`。你可以用一个类似`curl -XPOST localhost:8080`的命令来测试这个功能。
> 您可能需要使用`gcloud auth application-default`登录。

# 部署到云功能

运行以下命令将函数部署到云函数:

部署并测试我们的功能

我们已经将这个云功能设为私有，但是我们可以通过使用来自`gcloud`的认证令牌来测试这个云功能。工作流结果将出现在 HTTP 响应中。

```
Result: ["Wednesday","Wednesday Night Wars","Wednesday 13","Wednesday Campanella","Wednesday Addams","Wednesday Night Hockey (American TV program)","Wednesdayite","Wednesday Campanella discography","Wednesday Morning, 3 A.M.","Wednesday Theatre"]
```

如果有问题，我们还可以查看函数的 Stackdriver 日志。

# 感谢阅读

现在，您已经有了一个与云功能相关联的工作流，或许可以使用 Cloud Scheduler 为您的功能添加一个计划。你的思维是极限！

如果您喜欢这篇文章，我推荐您观看 YouTube 上关于云工作流的视频:

介绍 Google 云工作流

并在 GitHub 上查看我们所有的几十个工作流示例:

[https://github.com/GoogleCloudPlatform/workflows-samples](https://github.com/GoogleCloudPlatform/workflows-samples)