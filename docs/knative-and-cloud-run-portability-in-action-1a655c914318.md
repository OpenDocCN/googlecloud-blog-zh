# Knative 和 Cloud Run，将便携性付诸实践

> 原文：<https://medium.com/google-cloud/knative-and-cloud-run-portability-in-action-1a655c914318?source=collection_archive---------2----------------------->

![](img/baca99209bb3995df65a9fa8c3d80ea2.png)

[云跑](https://cloud.google.com/run/docs/)已经宣布将于 19 日在三藩市举行。一年前，Knative 已经由谷歌在同一事件中推出。两者都承诺同样的事情:**在任何安装了 [Istio](https://istio.io/) 和 [Knative](https://knative.dev/) 的 Kubernetes 集群**上的无服务器和可移植性。

在 Next19，我有机会站在舞台上宣布这个消息，演讲题目是: [**容器一次，无服务器随处**](https://www.youtube.com/watch?v=16vANkKxoAU)

> 无服务器的地方，真的吗？是真的吗？可能吗？

让我们验证这一点

# 云运行

Cloud Run 是这个承诺的服务名:完全符合 Knative 的 API，它有两种版本:托管的和基于 GKE 的。

*GKE 版本是 Anthos*[***云运行的一部分***](https://cloud.google.com/run/docs/gke/setup) *，它允许在 Anthos 管理的任何兼容集群中部署云运行服务:在 GKE 上，在本地，在其他云提供商上。*

## 托管版本

托管版本符合 Knative API，但是底层实现是特定于平台的。部署可以在控制台上完成，也可以通过命令行用`[gcloud](https://cloud.google.com/sdk/install)`完成。

```
gcloud beta run deploy hello --region us-central1 \
    --image gcr.io/cloudrun-hello-go/hello \
    --platform managed --allow-unauthenticated
```

几秒钟后，服务部署完毕，并显示可访问的 URL。测试一下！

```
curl https://hello-<project hash>.run.app/# Result
This created the revision hello-00001 of the Cloud Run service hello in the GCP project <project>
```

**好的**。让我们在 GKE 部署同样的东西

## GKE 版本的 Anthos

对于在 GKE 上的部署，您必须**使用 Cloud Run 插件**部署您的 GKE 集群。这个附加组件在 GKE 集群上安装所有需要的组件，比如 Istio 和 Knative。

让我们从[部署一个符合云运行](https://cloud.google.com/run/docs/gke/setup#create_cluster)的 GKE 集群开始。

```
gcloud beta container clusters create cloudrun-cluster\
--addons=HorizontalPodAutoscaling,HttpLoadBalancing,CloudRun \
--machine-type=n1-standard-2 \
--zone=us-central1-a \
--enable-stackdriver-kubernetes
```

然后用`gcloud`命令行部署服务

```
gcloud beta run deploy hello --cluster-location us-central1-a \
    --image gcr.io/cloudrun-hello-go/hello \
    --platform gke --connectivity external \
    --cluster cloudrun-cluster
```

考验的时候到了！但是**在 Kubernetes 集群上测试 Knative 服务不像使用托管版本**那么容易。有两件事要做:

*   用于获取外部 IP 的**入口网关**。这代表为接受入站流量而部署的负载平衡器。

```
kubectl get svc istio-ingressgateway --namespace istio-system \
   -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
```

*   **部署主机名**。*在部署结束时提供。*它用于将流量路由到正确的服务和 pod。

```
kubectl get route.serving.knative.dev hello \
    -o jsonpath='{.status.url}' | sed 's/http:\/\///g'
```

现在，将所有这些放在一起，像这样请求端点

```
curl -H "Host: $(kubectl get route.serving.knative.dev hello \
   -o jsonpath='{.status.url}' | sed 's/http:\/\///g')" \
   $(kubectl get svc istio-ingressgateway --namespace istio-system \
   -o jsonpath='{.status.loadBalancer.ingress[0].ip}')# Result
This created the revision hello-<hash> of the Cloud Run service hello in the GCP project <project>
```

伟大的**GKE 上的云运行和托管的云运行是兼容的**。*幸好(！！！)*，它们是 2 个 Google 云服务，使用了相同的 CLI 命令(对服务定义/特性做了一些更改)

# YAML 教的普遍性

Google 云环境的可移植性意味着**使用云提供商不可知的服务定义和部署**。

Kubernetes 使用`**YAML**` **文件格式向主节点**声明集群的所有元素。当然，Knative 也使用这种格式来定义服务。

## 最小 YAML

[主动发球](https://knative.dev/docs/serving/getting-started-knative-app/)有自己的发球定义。对于我们的测试，我定义了一个简单最小的`[hello.yaml](https://github.com/guillaumeblaquiere/cloudrun-hello-go/blob/master/hello.yaml)`文件。

```
apiVersion: serving.knative.dev/v1
kind: Service
metadata:  
  name: hello
spec:
  template:
    spec:
      containers:
      - image: gcr.io/cloudrun-hello-go/hello
```

*注:你会在* [*中找到很多例子*](https://knative.dev/docs/serving/getting-started-knative-app/#configuring-your-deployment)*`*metadata.namespace*`*的值集。它是可选的。如果未设置，则部署时使用当前的* `*namespace*` *，也可以用* `*--namespace*` *param 显式定义。**

## *云运行管理*

*2020 年初，谷歌在测试版**中发布了使用 YAML 文件来定义和部署服务**的功能(*和* *这是 Alpha 测试人员*反复提出的要求之一)。*

*使用`gcloud`命令行应用`yaml`。*对于云运行托管版本，没有其他方法可以与完全托管的集群进行交互。**

```
*gcloud beta run services replace hello.yaml --platform managed \
  --region us-central1* 
```

*该命令行应用`yaml`文件**来创建或更新(如果存在)服务**，但不允许定制访问模式。`replace`命令只是**应用** `**yaml**` **配置，而不与 Google IAM 服务**交互以允许未经授权的用户。顺便说一下，测试可以用以下两种方法进行:*

*   *通过在报头中添加 id 令牌来执行经过身份验证的请求*

```
*curl -H "Authorization: Bearer $(gcloud auth print-identity-token)"\
    https://hello-<project hash>.run.app/*
```

*   *允许所有未经身份验证的用户使用 IAM 命令，然后执行未经身份验证的请求*

```
*gcloud beta run services add-iam-policy-binding hello-yaml \
    --member allUsers --role roles/run.invoker --region us-central1curl https://hello-<project hash>.run.app/*
```

*结果是一样的*

```
*# Result
This created the revision hello-yaml-00001 of the Cloud Run service hello-yaml in the GCP project <project>*
```

*在**托管平台上，已经正确应用了** `**hello.yaml**` **文件，部署了**服务。继续使用其他平台*

## *GKE 和云一起奔向安索斯*

*回到安装了云运行附加组件的 GKE 平台。当然，谷歌的目标是**在托管平台和 GKE 平台上提供同样的体验**，同样的`replace`命令也适用于 GKE 集群。*

```
*gcloud beta run services replace hello.yaml --platform gke 
  --cluster-location us-central1-a --cluster k8s-cluster*
```

*但是这一次，**我不想粘着谷歌**。*这是云运行托管版本所必需的，因为我没有其他解决方案来与托管集群进行交互。**

*然而，这一次，我们有一个 Kubernetes 集群，我们可以使用专用工具。`**kubectl**` **是与 Kubernetes 集群交互的标准 CLI 工具。***

*Kubernetes 是一个声明性模型。由此，**我们将** `**apply**` **配置给主节点，它执行适当的动作**。*

```
*kubectl apply -f hello.yaml*
```

*现在，服务已经创建，为了测试它，我们可以重用以前的 GKE 命令行。*不要忘记将服务的名称改为* `*hello-yaml*`*

```
*curl -H "Host: $(kubectl get route.serving.knative.dev hello-yaml \
   -o jsonpath='{.status.url}' | sed 's/http:\/\///g')" \
   $(kubectl get svc istio-ingressgateway --namespace istio-system \
   -o jsonpath='{.status.loadBalancer.ingress[0].ip}')# Result
This created the revision hello-yaml-<hash> of the Cloud Run service hello-yaml in the GCP project <project>*
```

*完美！但是，**由于安装了 Cloud Run 附加组件，该集群 GKE 并不真正与 GCP 无关。我们能用更中性的东西吗？***

## *GKE 与 Istio 和 Knative*

*为此，我们将**部署一个没有安装 Google 插件的新集群**。*

```
*gcloud beta container clusters create k8s-cluster\
--machine-type=n1-standard-2 \
--zone=us-central1-a \
--enable-stackdriver-kubernetes*
```

*在安装 Knative 之前，需要 I [stio](https://istio.io/)*

```
*# [Download Istio](https://istio.io/docs/setup/#downloading-the-release)
curl -L https://git.io/getLatestIstio | ISTIO_VERSION=1.3.1 sh -# Go into directory
cd istio-1.3.1# [Install Istio](https://istio.io/docs/setup/install/kubernetes/)
for i in install/kubernetes/helm/istio-init/files/crd*yaml; \
    do kubectl apply -f $i; done# Activate the permissive mode (enough for the test)
kubectl apply -f install/kubernetes/istio-demo.yaml*
```

*然后，[在任何 Kubernetes 集群上安装 Knative](https://knative.dev/docs/install/knative-with-any-k8s/) 。*

*和以前一样，**使用 Kubectl 来应用服务配置***

```
*kubectl apply -f hello.yaml*
```

*最后，测试一下*

```
*curl -H "Host: $(kubectl get route.serving.knative.dev hello-yaml \
   -o jsonpath='{.status.url}' | sed 's/http:\/\///g')" \
   $(kubectl get svc istio-ingressgateway --namespace istio-system \
   -o jsonpath='{.status.loadBalancer.ingress[0].ip}')# Result
This created the revision hello-yaml-<hash> of the Cloud Run service hello-yaml in the GCP project <project>*
```

*太好了！但是，**还是在谷歌云环境下。**这些测试是否存在偏差？我们去别的地方测试吧。*

# *所有 kubernetes 集群？*

*这里是最后一个挑战:这个简单的 YAML 是否仍然符合谷歌云环境。为了验证这一点，我选择在亚马逊网络服务的 [EKS 产品上验证这一点。](https://aws.amazon.com/eks/)*

*我在 Medium 上跟随[这篇教程，部署一个 Kubernetes 集群需要大约 20 分钟的时间，还需要一些手动操作。您还可以使用](/faun/create-your-first-application-on-aws-eks-kubernetes-cluster-874ee9681293) [eksctl，通过一行 CLI](https://docs.aws.amazon.com/eks/latest/userguide/getting-started-eksctl.html) 来部署您的集群，这更加简单快捷，但不是通过 GUI。*

*当您的集群被部署，并且节点链接到您的主节点时，我的意思是当命令`kubectl get nodes`返回 3 行带有`READY`状态的代码时，**您可以用与之前在 GKE 上相同的方式安装 Istio 和 Knative** 。*

*和以前一样，**使用 Kubectl 来应用服务配置***

```
*kubectl apply -f hello.yaml*
```

*并且，测试一下。*

```
*curl -H "Host: $(kubectl get route.serving.knative.dev hello-yaml \
   -o jsonpath='{.status.url}' | sed 's/http:\/\///g')" \
   $(kubectl get svc istio-ingressgateway --namespace istio-system \
   -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')# Result
This created the revision hello-yaml-<hash> of the Cloud Run service hello-yaml*
```

****注意:****Istio Ingress Gateway 在 EKS 环境上提供了一个* `*hostname*` *，而不是像在 GKE 上那样提供一个* `*IP*` *。从一个提供者到另一个提供者，底层实现可能略有不同。**

# *无处不在的服务器*

***最后，是的**，相同的`hello.yaml`文件已经被部署在 4 个不同的 Knative 兼容环境中，服务以完全相同的方式工作。*

*当然，在这些示例中，为了更容易测试，**`**hello**`**容器是独立于平台的，并且允许避免集成约束**:不需要 IAM 角色来访问特定资源，如数据库、存储或其他无服务器产品。***

***不管怎么说，太牛逼了！Knative 项目非常棒，确保了很好的兼容性。带有`kubectl`的 Kubernetes CLI 命令对于部署和请求服务是相同的！**容器一次，无服务器随地”是真的，而便携性，成为现实！*****

***这是未来的良好基础。**事件、调试和其他重要的东西正在 Kubernetes、Knative 和 Cloud Run** 平台中到来。敬请期待，下一个版本将会很棒！***

****Github**上有* [*的源代码。代码是一个*](https://github.com/guillaumeblaquiere/cloudrun-hello-go) [*云-运行-你好*](https://github.com/GoogleCloudPlatform/cloud-run-hello) *的分支，但是没有 HTML 模板。****