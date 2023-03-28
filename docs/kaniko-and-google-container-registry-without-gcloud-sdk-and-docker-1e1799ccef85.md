# 没有 gcloud sdk 和 Docker 的 Kaniko 和 Google 容器注册表

> 原文：<https://medium.com/google-cloud/kaniko-and-google-container-registry-without-gcloud-sdk-and-docker-1e1799ccef85?source=collection_archive---------0----------------------->

今天我面临着一个挑战，我们在一个运行在 [Cloud Run](https://cloud.google.com/run) 上的小型微服务上为 Google Cloud 的 IAM 创建了一个访问令牌。用例是一个小的令牌交换服务，它允许从一个应用程序交换一个令牌来交换一个 IAM 访问令牌。

在下一步中，我想在我们的 [Kaniko](https://github.com/GoogleContainerTools/kaniko) 构建中使用这个令牌将一个 Docker 映像推送到我的项目的[容器注册表](https://cloud.google.com/container-registry)中。但是我如何向 Kaniko 提供访问令牌呢？由于我在其上运行构建的构建代理没有安装 Docker，这个问题变得更加糟糕。

通常我会调用 gcloud auth configure-docker。但是由于 gcloud 目前不支持直接提供访问令牌，所以我必须想出另一种方法来实现我的目标。

所以我构建了一个小的变通脚本，它创建了一个直接包含访问令牌的 Docker 配置，而不需要调用`gcloud auth configure-docker.`
。GCR 服务期望在**授权**头中提供的用户名和密码组合包含作为用户名的 gclouddockertoken 和作为密码的访问令牌。下面的脚本获取这些参数，base64 对它们进行编码，并将配置直接写入到`˜/.docker/config.json`中。

完成后，通过这个小的变通方法，您可以使用自己的令牌针对 GCR 进行身份验证，而不需要安装 Docker CLI 或 Docker 守护程序。
这种方法应该适用于 Docker 生态系统中依赖于 config.json 中的访问配置的许多工具。