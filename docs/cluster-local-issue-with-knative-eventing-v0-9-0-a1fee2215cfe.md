# 具有 Knative Eventing v0.9.0 的集群本地问题

> 原文：<https://medium.com/google-cloud/cluster-local-issue-with-knative-eventing-v0-9-0-a1fee2215cfe?source=collection_archive---------0----------------------->

在我之前的[帖子](/google-cloud/knative-v0-9-0-a6fa0a3b1f7d)中，我谈到了 Knative v0.9.0 和最新版本中的一些重大变化。从那以后，我一直在玩 Knative v0.9.0，用 [PullSubscription](https://github.com/google/knative-gcp/blob/master/docs/pullsubscription/README.md) 阅读谷歌云发布/订阅消息，我遇到了一个让我困惑了一段时间的基本问题。我想在这里概述一下问题和解决方案，以防对别人有用。

# 作为事件接收器的功能服务

在我的 PullSubscription 中，我可以将 Kubernetes 服务定义为如下事件接收器:

```
apiVersion: pubsub.cloud.run/v1alpha1
kind: PullSubscription
metadata:
  name: testing-source-event-display
spec:
  topic: testing
  sink:
    apiVersion: v1
    kind: Service
    name: event-display
```

这工作得很好，允许我在 Kubernetes 服务中阅读发布/订阅消息。下一步，我想使用 Knative 服务作为事件接收器，因为 Knative 服务更容易处理。在 PullSubscription 中，我将我的接收器定义如下:

```
apiVersion: pubsub.cloud.run/v1alpha1
kind: PullSubscription
metadata:
  name: testing-source-event-display
spec:
  topic: testing
  sink:
    apiVersion: serving.knative.dev/v1alpha1
    kind: Service
    name: event-display
```

我可以应用这个定义，并看到创建了与 PullSubscription 相关的 pod:

```
$ kubectl get podsNAME                 READY   STATUS
cre-pull-e7072664-ee7a-11e9-8ae9-42010a84006f-7db6dcf4b9-5ztgx    1/1     Running     2          7m12s
pubsub-s-testing-source-event-display-pullsubscription-cretw8mw   0/1     Completed   0          7m12s
```

然而，当我检查 pod 的日志时，我发现了一些错误:

```
$ kubectl logs cre-pull-e7072664-ee7a-11e9-8ae9-42010a84006f-7db6dcf4b9-5ztgx{"level":"info","ts":1571054849.2031848,"logger":"fallback","caller":"pubsub/transport.go:231","msg":"got an event!"}
{"level":"warn","ts":1571054849.2170587,"logger":"fallback","caller":"pubsub/transport.go:248","msg":"pubsub receiver return err","error":"Post [http://event-display.default.svc.cluster.local](http://event-display.default.svc.cluster.local): dial tcp: lookup event-display.default.svc.cluster.local on 10.0.16.10:53: no such host"}
```

PullSubscription 获取消息，但无法将其传递给 Knative 服务。`default.svc.cluster.local`暗示 Knative 期望我的服务是集群本地的(内部的)。以前不是这样的。正如我在教程中解释的那样，我试图将我的 Knative 服务转变为集群本地服务，但也没有成功。

# 集群本地问题

此时，我愣住了。经过研究，我意识到我并不孤单。有一个 bug ( [1973](https://github.com/knative/eventing/issues/1973) )其他用户也报告了类似的问题。感谢 Knative 工程团队和 bug 中的评论，我发现 Knative 现在需要一个额外的 Istio 集群本地网关添加到您的集群中，以便在 PullSubscription 中将 Knative 服务作为事件接收器。

# 安装集群本地网关

有一个[更新您的安装以使用集群本地网关](https://knative.dev/docs/install/installing-istio/#updating-your-install-to-use-cluster-local-gateway)页面，描述了如何将本地网关添加到您的集群。Knative Service 中还有一个 [third_party](https://github.com/knative/serving/tree/master/third_party) 文件夹，带有[istio-Knative-extras . YAML](https://github.com/knative/serving/blob/master/third_party/istio-1.2.7/istio-knative-extras.yaml)，您可以指向它来安装集群本地网关。

在我的例子中，我使用的是带有 Istio 插件的 GKE 集群，其版本如下:

*   GKE
*   Istio: 1.1.13-gke.0

截至今天，Knative 中的`third_party`文件夹有 Istio 版本`1.2.7`和`1.3.2`。与我的 Istio 版本不完全匹配，但我选择了`1.2.7`:

```
$ kubectl apply -f [https://raw.githubusercontent.com/knative/serving/master/third_party/](https://raw.githubusercontent.com/knative/serving/master/third_party/)istio-1.2.7/istio-knative-extras.yamlserviceaccount/cluster-local-gateway-service-account created
serviceaccount/istio-multi configured
clusterrole.rbac.authorization.k8s.io/istio-reader configured
clusterrolebinding.rbac.authorization.k8s.io/istio-multi configured
service/cluster-local-gateway created
deployment.apps/cluster-local-gateway created
```

在这之后，我能够使用 Knative 服务作为事件接收器。

这仍然不理想，因为我不确定随着 GKE 插件的升级，它是否能继续与不同版本的 Istio of 兼容。然而，它将作为一个临时的解决方案，直到我们拿出一个更完整的安装说明 Knative Eventing 对 GKE。

我在教程中概述了在 Knative Eventing 中接收发布/订阅消息所需的所有设置。在这里查看。