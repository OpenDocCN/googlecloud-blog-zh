# 。Google Cloud Buildpacks 和 Cloud Run 中的. NET 6 支持

> 原文：<https://medium.com/google-cloud/net-6-support-in-google-cloud-buildpacks-and-cloud-run-1b1d582b5f2d?source=collection_archive---------2----------------------->

我真的很喜欢容器的想法，一个我们的应用程序可以依赖的可复制的上下文。然而，我真的不喜欢创建一个`Dockerfile`来定义我的容器图像。我总是需要从某个地方复制&粘贴它，这花了我一些时间来得到它的权利。

这就是为什么我喜欢使用[谷歌云构建包](https://github.com/GoogleCloudPlatform/buildpacks)来构建容器，而不必担心`Dockerfile`。

**我很高兴地宣布，截至昨天，Google Cloud Buildpacks 已经支持。NET 6(最新的 LTS 版本)应用程序！**

这对你意味着什么？

这里有一个[。NET 6 Hello World app](https://github.com/meteatamel/cloudrun-dotnet-6) 带`Dockerfile`。

您可以在一个步骤中构建容器映像并部署到云运行:

```
gcloud run deploy $SERVICE_NAME \
 --source . \
 --allow-unauthenticated \
 --region $REGION
```

这使用了。NET buildpack 和`Dockerfile`来构建容器映像，然后部署到云运行:

```
This command is equivalent to running `gcloud builds submit --tag [IMAGE] .` and `gcloud run deploy dotnet6 --image [IMAGE] ` Building using Dockerfile and deploying container to Cloud Run service [dotnet6] in project [dotnet-atamel] region [europe-west1]
```

更好的是，如果你去掉`Dockerfile`。NET buildpack 检测。NET 版本并为您构建映像:

```
This command is equivalent to running `gcloud builds submit --pack image =[IMAGE] .` and `gcloud run deploy dotnet6 --image [IMAGE] ` Building using Buildpacks and deploying container to Cloud Run service [dotnet6] in project [knative-atamel] region [europe-west1]
```

这不仅适用于。NET 6 但是对于。NET 5 和所有的。以后的. NET 版本。网芯 1.0！

如有任何问题/反馈，请随时通过 Twitter [@meteatamel](https://twitter.com/meteatamel) 联系我。

*最初发布于*[*https://atamel . dev*](https://atamel.dev/posts/2022/02-04_dotnet6_buildpacks_cloudrun/)*。*