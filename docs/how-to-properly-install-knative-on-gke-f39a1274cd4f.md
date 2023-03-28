# 如何在 GKE 上正确安装 Knative

> 原文：<https://medium.com/google-cloud/how-to-properly-install-knative-on-gke-f39a1274cd4f?source=collection_archive---------1----------------------->

谷歌 Kubernete 引擎(GKE)的默认安装[指令](https://knative.dev/docs/install/knative-with-gke/)有问题(见 bug [2266](https://github.com/knative/eventing/issues/2266) )。在这篇文章中，我想概述一下问题是什么，告诉你我做了什么，并向你提供适合我的脚本，直到在 gcloud 或 Knative 中实现了一个适当的解决方案。

# 问题是

默认的 Knative 安装说明告诉您创建一个 GKE 集群，如下所示:

```
gcloud beta container clusters create $CLUSTER_NAME \
  --addons**=**HorizontalPodAutoscaling,HttpLoadBalancing,**Istio** \
  --machine-type**=**n1-standard-4 \
  --cluster-version**=**latest --zone**=**$CLUSTER_ZONE \
  --enable-stackdriver-kubernetes --enable-ip-alias \
  --enable-autoscaling --min-nodes**=1** --max-nodes**=10** \
  --enable-autorepair \
  --scopes cloud-platform
```

请注意 Istio 附加组件。这个命令创建一个已经安装了 Istio 的 Kubernetes 集群。这很好，因为 Istio 是 Knative 的依赖项，但请继续阅读。

之后，您继续安装 Knative Serving 和 Knative Eventing。在这一点上，Knative 服务将工作良好。然而，你会有 Knative Eventing 的问题。

正如我在之前的[帖子](/google-cloud/cluster-local-issue-with-knative-eventing-v0-9-0-a1fee2215cfe)中解释的，在 Knative Eventing 中，你不能使用 Knative 服务作为事件接收器。例如，你不能让 Knative 服务通过 Knative Eventing 接收 Google Cloud 发布/订阅消息(一个纯 Kubernetes 服务可以工作)。

为什么？因为您需要安装一个集群本地网关来将事件路由到 Knative 服务。Knative 文档有一个[页面](https://knative.dev/docs/install/installing-istio/#updating-your-install-to-use-cluster-local-gateway)解释如何安装它。但是，它已经过时了，依赖于 Helm。也许只是我，但是我不能使用那些指令使它工作。

# 初始无解

在我之前的[帖子](/google-cloud/cluster-local-issue-with-knative-eventing-v0-9-0-a1fee2215cfe)中，我提供了一个初步的解决方案，这个方案有点儿有效，但是现在不再有效了。概括地说，我谈到了 Knative Serving 中的一个第三方文件夹。在这个文件夹中，有几个版本的 Istio 文件夹(目前是 1.3.6 和 1.4.2)。在这些文件夹中，有一个`istio-knative-extras.yaml`文件，您可以应用它来安装集群本地网关。

**然而，这已经不起作用了，因为 GKE 的 Istio 附加版本远远落后于 1.3.6。** Knative 更新 Istio 版本的速度似乎比 GKE 插件更新的速度还要快。

# 我的解决方案

我的解决方案是基本上不再依赖 GKE Istio 附加软件。相反，我依赖于 Knative 的[第三方](https://github.com/knative/serving/tree/master/third_party)文件夹中的 Istio 版本，并使用其中的 yaml 文件安装我想要的 Istio 版本。同一个文件夹中还有用于安装集群本地网关的 yaml 文件。

## 创建没有 Istio 插件的 GKE 集群

```
gcloud beta container clusters create $CLUSTER_NAME \
--addons HorizontalPodAutoscaling,HttpLoadBalancing \
--zone $CLUSTER_ZONE \
--cluster-version $CLUSTER_MASTER_VERSION \
--machine-type $CLUSTER_NODE_MACHINE_TYPE \
--num-nodes $CLUSTER_START_NODE_SIZE \
--min-nodes 1 \
--max-nodes $CLUSTER_MAX_NODE_SIZE \
--enable-ip-alias \
--enable-stackdriver-kubernetes \
--enable-autoscaling \
--enable-autorepair \
--scopes cloud-platform
```

## 使用来自 Knative Serving 的 yaml 文件手动安装 Istio

```
ISTIO_VERSION=1.4.2kubectl apply -f https://raw.githubusercontent.com/knative/serving/master/third_party/istio-${ISTIO_VERSION}/istio-crds.yamlkubectl apply -f https://raw.githubusercontent.com/knative/serving/master/third_party/istio-${ISTIO_VERSION}/istio-minimal.yaml
```

## 安装群集本地网关

```
kubectl apply -f https://raw.githubusercontent.com/knative/serving/master/third_party/istio-${ISTIO_VERSION}/istio-knative-extras.yaml
```

# 剧本

因为我经常这样做，所以我用一个[设置](https://github.com/meteatamel/knative-tutorial/tree/master/setup)文件夹更新了我的 [Knative 教程](https://github.com/meteatamel/knative-tutorial)，在那里我有在 GKE 上正确设置 Knative 的脚本。如果你对如何改进它们有任何想法，请随时使用它们并给我发送请求。

# 病菌

我还在 Knative 上打开了这个的 bug ( [2266](https://github.com/knative/eventing/issues/2266) )。请投赞成票！