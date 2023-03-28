# 如何将 Docker 图像推送到 Google 私有容器注册表

> 原文：<https://medium.com/google-cloud/how-to-push-docker-image-to-google-private-container-registry-a2b58615e795?source=collection_archive---------0----------------------->

如果 docker 图像已经与我们在一起，我们想推到私人谷歌容器注册。[或者您可以构建自己的 docker 镜像 ref:[**Dockerimagebulid**](https://docs.docker.com/engine/reference/builder/)]

在将图像推送到 Google 容器注册表之前，您必须将注册表名称和图像名称作为标签添加到图像中。

**us.gcr.io 在美国托管你的图像。
eu.gcr.io 托管您在欧盟的图像。asia.gcr.io 托管您在亚洲的图像。**

## 将标签添加到您的图像:

*docker 标签<用户名> / <样本图像名称>gcr.io/<项目 id > / <样本图像名称> : <标签>*

## 然后，使用 gcloud 命令行工具将图像推送到 Google 容器注册表:

*gcloud docker —推送 gcr.io/your-project-id/<项目-id > / <样本-图像-名称> : <标签>*