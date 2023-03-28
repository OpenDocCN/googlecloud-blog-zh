# Google Cloud DevOps 系列——面向 Kubernetes 的 Google 云计算选项

> 原文：<https://medium.com/google-cloud/google-cloud-devops-series-4013adab603b?source=collection_archive---------0----------------------->

## Google Cloud DevOps 系列:第二部分

欢迎来到 Google Cloud DevOps 系列的第 2 部分..你可以在这里阅读该系列[的第一部分](/google-cloud/google-cloud-devops-part-1-introduction-to-google-native-devops-process-bfb55be9e3f3)

![](img/29186a6b8086d43b2cbcbb2a8cc0dfb2.png)![](img/a0e70e7aea524e51fde8f0407b6fc5ab.png)![](img/5f62a5e9405928331c7d6e2190f7a69f.png)![](img/6e07a08de53c97b6fef07870bf06ff1d.png)

**为什么是 GKE**

![](img/026999dbae296193f030862195da6c34.png)![](img/66bcc6438d757c35d4baa85a8a1c3a5b.png)![](img/a8890619a5f0ef673d518caacb8587d9.png)![](img/a7d0643148fee9a9e2a0d5033c45c96d.png)![](img/7757fd1f947c820535e72ac076f9f132.png)![](img/9d508623f43a2eed76a9228b7464d5cf.png)![](img/72f166c0ca2894cc2bb47db67335f8ec.png)![](img/62a35bb15a8d9d3bb43023da17843058.png)![](img/a23f441057cecf9a7e3058e2e27b36c6.png)![](img/8dc51e6f47807b5ab005209c8b1a9109.png)![](img/840623699a353eaf97847fcec19d0080.png)![](img/cc4bdec6d330e20c404eb3d4a29bc4b5.png)![](img/f9922c1e7734b3c9a2e54be57b7476d2.png)![](img/b074d088e3a724823b704dfbdaf6a85f.png)![](img/c48e6302d48975117f4ba310e63f0176.png)![](img/3742b89091d258decf4331894d1e3038.png)

**了解 GKE 自动驾驶**

Autopilot 是 GKE 的新部署选项，其中包括集群的整个底层基础架构，包括控制平面、节点和节点池。推荐给正在寻找真正免提管理的 Kubernetes 解决方案的客户。该服务会监控节点的健康状况及其容量，并自动进行必要的调整。该服务专为生产工作负载而设计，并针对最佳资源利用率进行了优化，同时控制成本。GKE 自动驾驶仪的一些主要特点是:

![](img/d8202cd31df5691741c78c83f9ff723a.png)

虽然标准部署模式在群集配置方面提供了更大的灵活性，但 Autopilot 最适合于对该服务提供的预配置设置感到满意的客户。有一些功能仅在标准模式部署中可用—可抢占的虚拟机支持、云 TPU、Istio、Windows 容器、二进制授权、容器威胁检测、节点选择/关联等。仅举几个例子。如果您的工作负载需要这些特性，标准模式可能更合适。建议在致电询问这种部署模式是否适合您的工作负载之前，先浏览一下 Autopilot 提供的全套功能以及此处[解释的限制](https://cloud.google.com/kubernetes-engine/docs/concepts/autopilot-overview)。

![](img/ec7483432780f548d43679108710dec2.png)

**GKE 入门**

让我们从 GKE 开始，创建一个集群并在其中部署一个示例应用程序。下面给出的步骤可以从 Google Cloud Shell 中执行。演示应用程序可在以下 GitHub repo 中获得:[https://github.com/GoogleCloudPlatform/microservices-demo](https://github.com/GoogleCloudPlatform/microservices-demo.)。

**先决条件:**确保在您的 GCP 项目中启用了 GKE 和云操作 API。

首先设置项目 id

```
PROJECT_ID=”<your-project-id>"
```

启用 GKE API

```
gcloud services enable container.googleapis.com — project ${PROJECT_ID}
```

启用云操作 API

```
gcloud services enable monitoring.googleapis.com \
cloudtrace.googleapis.com \
clouddebugger.googleapis.com \
cloudprofiler.googleapis.com \
— project ${PROJECT_ID}
```

1.  首先使用下面的命令设置项目 ID:

```
gcloud config set project <<project id>
```

![](img/12d304ef55c437ce1cdf761d4e1384dd.png)

2.为群集设置计算区域。在本演示中，我们将使用区域 us-central1-a

```
gcloud config set compute/zone us-central1-a
```

![](img/b48d72e75cc068ce216d3cb5f2dc7c7a.png)

3.现在，让我们创建一个名为“testcluster”的 GKE 标准集群，使用默认设置，在启用了自动扩展的节点池中有一个节点

```
gcloud container clusters create hello-cluster — enable-autoscaling — min-nodes=1 — max-nodes=3
```

![](img/47e2609577c6d6db4b01e06cf922f4eb.png)

4.克隆微服务演示应用程序

```
git clone [https://github.com/GoogleCloudPlatform/microservices-demo.git](https://github.com/GoogleCloudPlatform/microservices-demo.git)cd microservices-demo
```

![](img/2b9aa0f7997c7b3a9b7c1ea6e7ed320b.png)

5.将示例应用程序部署到 GKE 集群

```
kubectl apply -f ./release/kubernetes-manifests.yaml
```

![](img/30731b4533008e5ba6d8809493eee7ab.png)

6.一旦部署成功完成，您应该能够使用以下命令看到与应用程序相关联的窗格

```
kubectl get pods
```

![](img/ce0d981e072246f6a3036e2766d8ac2c.png)

7.使用端口 80 上的外部 IP 地址访问应用程序

```
kubectl get service frontend-external | awk ‘{print $4}’
```

![](img/947ff9901eaa75c86e216acf137b351f.png)

或者，您也可以在 GCP 门户->服务和入口中浏览到 Kubernetes 引擎服务，并查找名为“前端-外部”的服务端点。

单击链接后，应用程序将会打开:

![](img/eb8c6191a306d0863594897665fb9437.png)![](img/5a78af6f4f6708d8c704de5d16b8880f.png)

在这一步中，我们从云外壳部署了集群，但是，包括集群创建和应用部署在内的整个过程都可以使用 DevOps 工具实现自动化。我们将在后面详细探讨这一点。

![](img/e7db504c844f458507abe462c83027b2.png)

**无服务器容器**

除了 GKE，您还可以使用 Cloud Run 以无服务器的方式部署您的容器。基于开源 Knative 标准，Cloud Run 抽象出基础设施管理的复杂性，使您能够专注于在自己选择的平台上开发应用程序。云运行支持多种流行语言如 Go、Python、Java、Ruby、Node.js 等。因为它是基于 Knative 的，所以它确保了应用程序到其他兼容环境的可移植性。

Cloud Run 可以部署为完全托管的无服务器解决方案，也可以通过 Cloud Run for Anthos 服务使用现有的 GKE 部署。后者充当 Anthos 集成，可用于将 pod 部署到本地以及使用无服务器结构的多云 K8s 集群。对于在 GKE 已有投资的组织来说，该服务将加快无服务器部署。

**接下来…**

在这篇博客中，我们了解了 Google Cloud 中 Kubernetes 的各种计算选项。听完这些细节后，古汉开始考虑 Samajik 的开发人员在 Google Cloud for containers 上开始 CI/CD 工作流的选项。让我们继续关注他与拉姆的(持续)对话。

撰稿人:[丹杜斯](https://medium.com/u/71d9487165c6?source=post_page-----4013adab603b--------------------------------)、[安其特·尼尚特](https://medium.com/u/2d47f7f3f8e2?source=post_page-----4013adab603b--------------------------------)、[普什卡尔·科塔瓦德](https://medium.com/u/c79cc28e2999?source=post_page-----4013adab603b--------------------------------)、[图沙尔·古普塔](https://medium.com/u/ee905ea343d?source=post_page-----4013adab603b--------------------------------)

更新:你可以在这里阅读 [Part-3](/google-cloud/part-3-google-devops-continuous-development-workflow-3ef446edfeb7) 。