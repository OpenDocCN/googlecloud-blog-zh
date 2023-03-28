# Anthos 的云运行事件> = Kubernetes 上的 Knative 事件

> 原文：<https://medium.com/google-cloud/events-for-cloud-run-for-anthos-knative-eventing-on-kubernetes-9966f44a4cad?source=collection_archive---------0----------------------->

# 介绍

我们最近[宣布了](https://cloud.google.com/blog/products/serverless/cloud-run-for-anthos-adds-events)一项新功能，*Events for Cloud Run for Anthos*，在 Google Kubernetes Engine (GKE)上构建事件驱动系统。在公告中，我们还声明该解决方案是基于开源的 [Knative](https://knative.dev/) 。

在这篇博文中，我想进一步解释这个新特性和 Knative 之间的关系。我还想让你相信，我们的解决方案是在 Google Cloud 上部署 Knative 兼容事件消费服务的一种更简单的方式。

**TLDR:*Events for Cloud Run for Anthos*是**[**Knative Events**](https://knative.dev/docs/eventing/)**为 Google Cloud 打包简化。**

当然，您仍然可以创建一个 GKE 集群，安装 Istio，在上面安装 Knative，并自行部署读取 Knative 事件的服务。然而，这将花费你更长的时间来设置一切，更不用说跟踪所有版本和依赖项的维护工作了。

让我们通过一个例子来深入了解细节。假设您想要部署一个服务来从 Google 云存储桶中读取事件。在 Knative vs. Cloud Run for Anthos 中设置此功能需要什么？

# 破坏性事件

## 创建启用事件的集群

要使用 Knative eventing 读取 Google 云存储事件，首先，您需要一个启用 Knative 的集群，并且您需要安装事件源来读取 Google Cloud 事件。大致步骤如下:

1.  创建 GKE 集群。
2.  安装 Istio。
3.  安装有创意的服务。
4.  安装 Knative Eventing。
5.  用 GCP 组件安装和配置 Knative。

我实际上试图在我的 [Knative 教程](https://medium.com/r?url=https%3A%2F%2Fgithub.com%2Fmeteatamel%2Fknative-tutorial)中维护一个[设置脚本](https://github.com/meteatamel/knative-tutorial/tree/master/setup)来做所有这些，每次有新版本的 Knative 时都要更新这个脚本，这是一个很大的工作量。

## 部署服务并连接到事件

拥有集群后，您需要:

1.  创建一个 CloudStorageSource，将云存储事件读入集群。
2.  在名称空间中注入一个代理来接收事件。
3.  部署一个 Knative 服务来消费事件。
4.  创建一个触发器来过滤云存储事件，并将其从代理传递到 Knative 服务。

我不会用细节来烦你，在我的实用教程中有一个[云存储触发的服务示例](https://github.com/meteatamel/knative-tutorial/blob/master/docs/storageeventing.md),里面有详细的说明，包括多个 yaml 文件、部署等等。绝对不是一个快速简单的过程。

# Anthos 的云运行事件

现在，让我们为 Anthos 创建相同的云运行示例。

## 创建启用事件的集群

像以前一样，我们需要一个支持事件的 GKE 集群。这是一个分两步走的过程。

在**快速**频道上创建一个带有 **CloudRun 附加组件**的 GKE 集群:

```
gcloud beta container clusters create ${CLUSTER_NAME} \
  --addons=HttpLoadBalancing,HorizontalPodAutoscaling,CloudRun \
  --machine-type=n1-standard-4 \
  --enable-autoscaling --min-nodes=3 --max-nodes=10 \
  --no-issue-client-certificate --num-nodes=3 --image-type=cos \
  --enable-stackdriver-kubernetes \
  --scopes=cloud-platform,logging-write,monitoring-write,pubsub \
  --zone ${CLUSTER_ZONE} \
  --release-channel=rapid
```

使用事件组件初始化集群:

```
gcloud beta events init
```

此时，您将拥有一个启用了事件的集群，所有设置都已完成。没有 Istio 安装，没有 Knative 安装，没有 Knative-GCP 安装和配置！

## 部署服务并连接到事件

这是一个与 Knative Eventing 相似但稍微简化的过程，包括代理、触发器等。

首先，在名称空间中注入一个代理:

```
gcloud beta events brokers create default --namespace default
```

将云运行服务部署为事件消费者:

```
gcloud run deploy event-display \
   --image gcr.io/knative-releases/knative.dev/eventing-contrib/cmd/event_display@sha256:8da2440b62a5c077d9882ed50397730e84d07037b1c8a3e40ff6b89c37332b27
```

创建一个触发器来过滤云存储事件，并将其从代理传递到云运行服务:

```
gcloud beta events triggers create trigger-storage \
  --target-service event-display \
  --type=google.cloud.storage.object.v1.finalized \
  --parameters bucket=bucket-name
```

注意，我们不必手动创建一个`CloudStorageSource`，触发器已经为我们创建了它。我们不必处理 yaml 文件，而是依靠 gcloud 命令来为我们做正确的事情。最重要的是，我们不必手动升级 Knative Eventing 前进，GKE 运营商可以为我们做到这一点！

希望这篇文章澄清了 Anthos 云运行的 Knative 事件和*事件之间的关系。*

要了解更多信息:

*   参见[快速入门](https://cloud.google.com/run/docs/events/anthos/quickstart)。
*   尝试 codelab: [为 Anthos 运行云的事件](https://codelabs.developers.google.com/codelabs/cloud-run-events-anthos/)。
*   尝试 qwiklab: [为 Anthos 构建一个包含云运行事件的 BigQuery 处理管道](https://google.qwiklabs.com/focuses/14220?parent=catalog)。

欢迎在 Twitter [@meteatamel](https://twitter.com/meteatamel) 上联系我，或者阅读我以前在 [medium/@meteatamel](/@meteatamel) 上的帖子。

*原载于*[*https://atamel . dev*](https://atamel.dev/posts/2020/10-09_events_cloud_run_anthos_knative_eventing/)*。*