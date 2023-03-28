# 无服务器 CI/CD — Google 云构建

> 原文：<https://medium.com/google-cloud/serverless-ci-cd-google-cloud-build-5b0c92d95d14?source=collection_archive---------0----------------------->

正如你从[我以前的博客](/google-cloud/updated-zero-to-continuous-delivery-with-gcp-10a1de80454c)中看到的，我是[持续部署](https://www.atlassian.com/continuous-delivery/ci-vs-ci-vs-cd)的忠实粉丝。一般来说，我希望利用开源平台来提供工具，以帮助构建和部署应用程序，工具如[詹金斯](https://jenkins.io/)，特别是 [Kubernetes 插件，用于基于临时容器的构建代理](https://github.com/jenkinsci/kubernetes-plugin)。

但是……我也热衷于允许企业通过尽可能利用平台( [PaaS](https://en.wikipedia.org/wiki/Platform_as_a_service) 或类似平台)来专注于他们的核心价值主张。利用技术解决方案意味着企业可以让员工专注于关键功能，让运营团队专注于管理平台的核心，让云提供商管理其余部分。

CI/CD 工具是许多组织需要的基础设施之一(特别是如果他们希望审查部署质量或部署节奏)，但管理这些“内部感觉”系统可能会占用您的系统管理员/操作员/ SRE 员工的一些时间。

在 [Google Cloud Next 2018](https://cloud.withgoogle.com/next18/sf/) 期间，谷歌宣布了其[无服务器产品](https://cloud.google.com/serverless/)中的大量产品/功能。我决定尝试的第一个工具是 [Google Cloud Build](https://cloud.google.com/cloud-build/) 。

本文的剩余部分提供了一个使用 Google Cloud Build 构建 NodeJS express 应用程序的运行示例。

TL；DR —本文附带的 GitHub repo

[](https://github.com/clos-consultancy/gcp-cloud-build) [## clos-咨询/GCP-云构建

### 博客的 GCP 云构建示例。通过创建以下帐户，为 clos-consultancy/GCP-cloud-build 开发做出贡献…

github.com](https://github.com/clos-consultancy/gcp-cloud-build) 

首先，我创建了一个真正标准的 NodeJS express 应用程序。为了运行单元测试，我决定利用 [Jest](https://jestjs.io/) 。

这是应用程序的 package.json

那里没什么特别的。

此外，对于应用程序，我们没有利用任何谷歌云工具来编写我们的应用程序。让我们的应用保持良好的平台无关性。

我非常喜欢代码定义的构建管道。Google Cloud Build 现在有所不同，它使用了一个名为 [cloudbuild.yaml](https://cloud.google.com/cloud-build/docs/configuring-builds/create-basic-configuration) 的文件——Cloud Build . YAML 文件告知 Google 构建/测试/部署应用程序的步骤。

下面是 Node 应用程序的初始构建文件示例。

非常简单——我们使用 Google Cloud 构建来安装我们的依赖项，然后再次使用 npm cloud 构建来运行我们的测试。

然后我们可以利用 GCP 命令行[提交这个构建到谷歌云](https://cloud.google.com/cloud-build/docs/running-builds/start-build-manually)。

```
gcloud builds submit --config cloudbuild.yaml .
```

注意:上面的命令假设您已经配置了 GCP 命令行和目标项目。如果您还没有在云项目上启用云构建 API，也不用担心，因为命令行会询问您是否希望启用它。

一旦你提交了你的构建，你就可以访问 c [音箱并查看你的构建日志](https://console.cloud.google.com/cloud-build/builds)。完全无服务器构建，无需监控/维护构建代理！

当然手动提交只能让你到此为止——如果你想自动化，你当然可以写你自己的 git 挂钩，但是 GCP 也提供[构建触发器](https://cloud.google.com/cloud-build/docs/running-builds/automate-builds)允许你基于 Git 变化自动触发构建。

当然，对于所有云产品，他们都有基于 op-ex 的费用—在编写云构建时，每天提供 120 分钟的免费构建时间。[完整的定价详情](https://cloud.google.com/cloud-build/pricing)

对于我们开发和支持的应用程序来说，这只是我们需要考虑的又一个工具。

有兴趣听听人们对使用无服务器 CI/CD 工具的看法。