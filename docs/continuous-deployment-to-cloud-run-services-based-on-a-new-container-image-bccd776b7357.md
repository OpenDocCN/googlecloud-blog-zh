# 基于新的容器映像持续部署到云运行服务。

> 原文：<https://medium.com/google-cloud/continuous-deployment-to-cloud-run-services-based-on-a-new-container-image-bccd776b7357?source=collection_archive---------0----------------------->

![](img/ed66cd5ed2d6eb4e8dd8445e3ecd5aa0.png)

> *本文介绍了一种基于新容器映像持续部署云运行服务修订版的方法。*

# 持续部署

持续部署是软件发布的一种策略，其中任何通过自动化测试阶段的代码提交都被自动发布到生产环境中，做出对软件用户可见的更改。

# 云运行

Cloud Run 是一个托管计算平台，使您能够在其无服务器环境中运行无状态容器，并抽象出所有基础架构管理，因此您可以专注于最重要的事情—构建优秀的应用程序。

如果你是云运行的新手，可以在这里随意阅读它的文档[。](https://cloud.google.com/run/)

**基于一个容器映像跨多个未知服务在云上连续部署可能会很有挑战性。**

> *当您将新图像推送到标签引用时，Google Cloud Run 不会自动部署修订。有很多很好的理由说明它不会。*
> 
> *部署云运行修订版时，它会计算映像引用的 sha256 哈希。*
> 
> *因此，当您使用:latest 标记指定容器映像时，Cloud Run 使用其 sha256 引用来部署和扩展您服务的修订版。当您更新:latest 标记以指向新图像时，云运行仍将使用以前的图像。否则，这将是一个危险的滑坡。—* [*阿赫迈特上 SO*](https://stackoverflow.com/a/56919456/4620227)

# 单一云运行服务的持续部署

Google Cloud 构建触发器可用于在代码更新时构建和推送新的容器映像，您还可以在构建配置中添加另一个步骤，以实现持续部署。

这是一个云构建配置的例子，它构建新的图像并将其推送到谷歌容器注册中心(GCR)，它还部署了一个新版本的云运行服务**仪表板**。

```
# cloudbuild.yaml
# Build and Push New Image to Google Container Registry
- name: "gcr.io/kaniko-project/executor:latest"
  args: ["--cache=true", "--cache-ttl=48h", "--destination=gcr.io/project/dashboard:latest"]# Extra step to Deploy New Revision to Cloud Run
- name: "gcr.io/cloud-builders/gcloud"
  args: ['beta', 'run', 'deploy', 'dashboard', '--image', 'gcr.io/project/dashboard:latest', '--region', 'us-central1', '--allow-unauthenticated', '--platform', 'managed']
```

# 多个云运行服务的持续部署

如果您需要更新多个云运行服务，只要您知道每个云运行服务的服务名称，就可以在您的云构建配置中有额外的步骤来部署云运行服务。

> *在*[*Mercurie*](https://mercurie.ng)*，我们将云运行用于多租户—这涉及到为每个新客户端(或租户)自动创建新服务。*

为了实现跨商店的连续部署，您可以编写一个云函数，用更新后的容器映像将新的版本部署到您的多个云运行服务。其工作原理是通过发布/订阅订阅云构建的通知，然后触发云功能。

# 更新云运行服务的云功能

云函数允许用户编写一次性的编程函数来监听云事件，比如构建。当云事件发生时，触发器通过执行一个动作来响应，例如发送云发布/订阅消息。

当您的构建状态改变时，例如当您的构建被创建时，当您的构建转换到工作状态时，以及当您的构建完成时，Cloud Build 在 Google Cloud Pub/Sub 主题上发布消息。

Cloud Build 将这些构建更新消息发布到的发布/订阅主题称为 **cloud-builds** ，当您启用 Cloud Build API 时，它会自动为您创建。每条消息都包含构建资源的 JSON 表示，消息的属性字段包含构建的惟一 ID 和构建的状态。

访问[云功能](https://console.cloud.google.com/functions/)和*创建功能*

*   输入您的函数名: **updateCloudRunServices**
*   设置*内存分配* : **256MB**
*   设置*触发* : **云发布/订阅**
*   设置*主题* : **云构建**
*   *源代码*:选择— **行内编辑器**
*   *运行时* : **Node.js 8**
*   *执行*的函数: **updateCloudRunServices**

将以下代码片段粘贴到内联编辑器中，并创建您的函数。

您现在有了一个云函数，它在源代码更新时监听来自云构建的发布/订阅通知，这还会构建一个新的容器映像，并使用云构建触发器推送到 GCR。

它检查更新是否针对**仪表板**源代码回购，以及构建状态是否为**成功**，然后调用**updateDashboardRevisions**方法，该方法使用**令牌**向云运行发出 API 请求，获取所有服务，使用容器映像[**【gcr.io/project/dashboard:latest】**](https://hashnode.com/util/redirect?url=http://gcr.io/project/dashboard:latest)过滤它们以仅获取服务。

然后创建包含步骤的新构建配置，并通过其 API 提交给云构建。这成功地更新了所有基于我们的容器映像[](https://hashnode.com/util/redirect?url=http://gcr.io/project/dashboard:latest)**的云运行服务。**

## **额外资源**

*   **[谷歌云运行](https://cloud.google.com/run/)**
*   **[谷歌云构建](https://cloud.google.com/cloud-build)**
*   **[谷歌云功能](https://cloud.google.com/functions)**
*   **[谷歌云发布/订阅](https://cloud.google.com/pubsub/)**
*   **[牛逼的谷歌云平台](https://github.com/GoogleCloudPlatform/awesome-google-cloud)**

> ***感谢通读！如果我错过了任何步骤，如果有些事情不太适合你，或者如果这个指南有帮助，请告诉我。***