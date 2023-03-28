# Windows 上的 Istio + Kubernetes

> 原文：<https://medium.com/google-cloud/istio-kubernetes-on-windows-f0666b126286?source=collection_archive---------1----------------------->

![](img/a4148279e22772853ddab62b69954c40.png)

我最近一直在研究 [Istio](https://istio.io/) ，一个连接和管理微服务的开放平台。在 Containers 和 Kubernetes 之后，我相信 Istio 是我们微服务之旅的下一步，我们将在如何管理和保护微服务的工具和方法上实现标准化。自然，我非常兴奋能得到 Istio。

虽然在 Google Kubernetes Engine (GKE)上设置 Istio 非常简单，但是在本地进行调试和测试总是很有用的。我特别想在我的 Windows 机器上的本地 [Minikube](https://github.com/kubernetes/minikube) Kubernetes 集群上安装 Istio。我遇到了一些小问题，我想在这里概述一下，以防对其他人有用。

我假设您已经安装并运行了一个 Minikube 集群。如果没有，你可以看看我在[之前发表的关于如何在你的 Windows 机器上设置和运行 Minikube 集群的文章](https://meteatamel.wordpress.com/2018/02/14/minikube-on-windows/)。Istio 有一个针对 Kubernetes 的[快速入门教程](https://istio.io/docs/setup/kubernetes/quick-start.html)。我将遵循这一点，但它是以 Linux 为中心的，一些命令必须被 Windows 采用。

# 下载 Istio

下面是从 Quickstart 下载 Istio 的命令:

```
curl -L [https://git.io/getLatestIstio](https://git.io/getLatestIstio) | sh -
```

这是一个 Linux shell 命令，它不能在 Windows cmd 或 PowerShell 上运行。谢天谢地，已经有人在这里写了一个等价的 PowerShell 脚本[。我按原样使用了脚本，只将*isti osition*更改为 *0.5.1* ，这是截至今天的最新 Istio 版本:](https://gist.github.com/kameshsampath/796060a806da15b39aa9569c8f8e6bcf)

```
param( [string] $IstioVersion = "0.5.1" )
```

该脚本下载 Istio 并将一个 *ISTIO_HOME* 设置为环境变量。

```
PS C:\dev\local\istio> .\getLatestIstio.ps1 Downloading Istio from [https://github.com/istio/istio/releases/download/](https://github.com/istio/istio/releases/download/) 0.5.1/istio_0.5.1_win.zip to path C:\dev\local\istio
```

然后，我将 *%ISTIO_HOME%\bin* 添加到*路径*中，以确保我可以运行 *istoctl* 命令。

# 安装并验证 Istio

为了安装 Istio 并启用边车之间的相互 TLS 认证，我在[快速入门](https://istio.io/docs/setup/kubernetes/quick-start.html)中运行了相同的命令:

```
PS C:\istio-0.5.1> kubectl apply -f install/kubernetes/istio-auth.yaml namespace "istio-system" created clusterrole 
"istio-pilot-istio-system" created clusterrole 
"istio-sidecar-injector-istio-system" created clusterrole 
"istio-mixer-istio-system" created clusterrole 
"istio-ca-istio-system" created clusterrole 
"istio-sidecar-istio-system" created
```

并验证所有 Istio pods 都在运行:

```
PS C:\istio-0.5.1> kubectl get pods -n istio-system NAME READY STATUS RESTARTS AGE 
istio-ca-797dfb66c5-x4bzs 1/1 Running 0 2m 
istio-ingress-84f75844c4-dc4f9 1/1 Running 0 2m 
istio-mixer-9bf85fc68-z57nq 3/3 Running 0 2m 
istio-pilot-575679c565-wpcrf /2 Running 0 2m
```

# 部署示例应用程序

在 Windows 上部署应用程序也略有不同。要使用 Envoy 容器注入部署 BookSample 应用程序，您通常会在 Linux 上运行以下命令:

```
kubectl create -f <(istioctl kube-inject -f samples/bookinfo/kube/bookinfo.yaml)
```

重定向会导致 PowerShell 出现问题。相反，您可以首先运行 *istioctl* 命令，并将其保存到一个中间 yaml:

```
istioctl kube-inject -f .\samples\bookinfo\kube\bookinfo.yaml > bookinfo_inject.yaml
```

然后，您可以应用中间 yaml:

```
PS C:\istio-0.5.1> kubectl create -f .\bookinfo_inject.yaml service "details" created 
deployment "details-v1" created 
service "ratings" created 
deployment "ratings-v1" created 
service "reviews" created 
deployment "reviews-v1" created 
deployment "reviews-v2" created 
deployment "reviews-v3" created 
service "productpage" created 
deployment "productpage-v1" created 
ingress "gateway" created
```

这样，您将拥有由 Istio 部署和管理的 BookInfo 应用程序。希望这有助于让 Istio + Kubernetes 在 Windows 上的 Minikube 中运行。

*原载于 2018 年 2 月 19 日*[*【meteatamel.wordpress.com】*](https://meteatamel.wordpress.com/2018/02/19/istio-kubernetes-on-windows/)*。*