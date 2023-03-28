# 如何在 Google Cloud 上运行无服务器批处理作业

> 原文：<https://medium.com/google-cloud/how-to-run-serverless-batch-jobs-on-google-cloud-ca45a4e33cb1?source=collection_archive---------1----------------------->

## 对于耗时超过几分钟的功能，使用人工智能平台

云函数、云运行、AppEngine 等。对于长时间运行的功能来说并不是一个好的选择，也就是说，任何运行时间超过几分钟的功能(服务本身限制在 10 到 15 分钟，但是包括错误和重试，所以您的目标应该是最多 2 到 3 分钟)。如果你想运行一个比这个时间更长的函数，你有什么选择？

![](img/fd9516f75ab0ea37987b5abddffcdba4.png)

如果您想以无服务器方式运行长时间运行的批处理作业，该怎么办？

把你的代码放在 Docker 容器中。使用人工智能平台运行它。使用云调度程序对其进行调度。

## 人工智能平台培训中的定制容器

你可以使用 AI 平台训练[运行任何任意 Docker 容器](https://cloud.google.com/ml-engine/docs/custom-containers-training)——不一定是机器学习的工作。要在 GPU 上执行任意容器，您只需:

```
gcloud ai-platform jobs submit training gpu_function \
  --scale-tier BASIC_GPU \
  --region $REGION \
  --master-image-uri gcr.io/$PROJECT_ID/some-image-name
```

这只是一个 REST API，所以您可以从许多编程语言中的各种客户端库调用它。对容器没有任何要求——只是它需要有一个入口点，并且在容器注册表中发布。可以使用定制的机器类型——详见[文档](https://cloud.google.com/ml-engine/docs/custom-containers-training)。

## 并发自动缩放？

能够在特定于作业的集群上启动自定义容器满足了许多无服务器功能的用例。但不是全部。具体来说，无服务器功能的另一个用例是并发自动扩展—我们希望能够接收多个请求，并将它们路由到同一台机器，一旦该机器开始不堪重负，我们希望添加更多的机器。如果您需要并行自动缩放，并且您的任务持续时间超过 2-3 分钟，那么 AI 平台培训解决方案将不起作用。在这种情况下，您将需要 Kubernetes，并且它不会是无服务器的。