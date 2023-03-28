# 四向容器

> 原文：<https://medium.com/google-cloud/containers-four-ways-457f4b7dd898?source=collection_archive---------0----------------------->

在当地的一次聚会上，我参加了集装箱会议，集装箱化是一个相对常见的讨论话题。谷歌是容器的大力倡导者，所以在谷歌云平台上有四种不同的方式来运行容器可能并不奇怪。

## Kubernetes 和 Google 容器引擎

在谷歌云上运行容器最明显的方式是使用 [Kubernetes](http://kubernetes.io) 和[谷歌容器引擎](https://cloud.google.com/container-engine/)。Kubernetes 是一个开源的容器编排工具，始于 Google。我喜欢你在使用 Kubernetes 时描述健康的跑步系统应该是什么样子。你写出描述你想要什么样的容器组(pod)，每组需要多少，以及它们应该如何相互通信的文件。Kubernetes 在虚拟机上调度容器，并负责联网。

当您的系统有多个具有不同伸缩特性的组件时，Kubernetes 和 Container Engine 是一个不错的选择。例如，您可能有一个身份验证服务和一个数据处理服务。使用 Kubernetes，您可以轻松适应这些服务的不同伸缩特性。您可以将运行每个服务的容器放在不同的 pod 中，并为每种类型的 pod 创建不同的自动缩放规则。Kubernetes 为您提供了自动伸缩逻辑和一定程度的自动修复的优势。如果托管您的容器的虚拟机出现故障，Kubernetes 会在剩余的主机上自动重新调度在该虚拟机上运行的容器。最后，Kubernetes 支持滚动更新，如果您在应用程序上实践连续部署，这是非常好的。

## App Engine Flex

如果你只有一个容器，而这个容器运行一个 web 应用，那么 [App Engine Flex](https://cloud.google.com/appengine/docs/flexible/) 是一个运行它的好方法。App Engine Flex 为您提供了 App Engine 的许多优势(自动扩展、自动修复、集成诊断和简单部署),但您不会受限于 App Engine 过去使用的受限运行时。相反，每种受支持的语言都有基于 Dockerfile 的基本图像。如果现有的容器映像不适合您，那么您可以通过使用[自定义运行时间](https://cloud.google.com/appengine/docs/flexible/custom-runtimes/)来使用您自己的容器映像。

App Engine Flex 仍处于测试阶段，但如果您的应用程序运行一个网站或 web 后端，并且您希望在一个完全托管的平台上运行它，那么它是值得一看的。使用 App Engine Flex，您可以获得基于容器的平台的所有优势(快速启动时间和可移植性),但如果您不想编写 docker 文件，则不必自己编写。Flex 也非常支持无缝更新你的应用程序。

## 托管实例组的容器映像

如果只需要运行一个容器，还可以使用在托管实例组上运行的容器映像。[托管实例组上的容器映像](https://cloud.google.com/compute/docs/instance-groups/deploying-docker-containers)。此功能目前处于 Alpha 版本，因此您必须请求将其添加到白名单中才能使用。

要使用这种方法在 GCP 上运行容器，您必须编写自己的 docker 文件，构建它，并将其推送到 Google 容器注册表。您也可以使用 Docker Hub 的公开图片。这种运行容器的方式非常适合不适合网站的容器工作负载，比如数据处理。当您希望完全控制容器在虚拟机上的分布时，这也是一个不错的选择。它既支持在每个虚拟机上运行单个容器，也支持运行多个容器。它目前不支持滚动更新，这是它不适合网站的另一个原因。

## 谷歌计算引擎

在 Google 云平台上运行容器的最后一种方法是在普通的旧计算引擎虚拟机上运行容器。以这种方式运行容器可以让您更好地控制它们的运行方式。但是您必须自己安装运行容器的所有基础设施。如果你需要非常严格地控制你的容器如何运行，或者你不准备转移到一个更受管理的工具，这是在 GCP 上运行容器的另一个选择。

## 结论

这是在谷歌云平台上运行容器的四种不同方式。希望有一个适合你。