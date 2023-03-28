# 通过 Docker 使用 Google 的私有容器注册表

> 原文：<https://medium.com/google-cloud/using-googles-private-container-registry-with-docker-1b470cf3f50a?source=collection_archive---------0----------------------->

# **概述**

通过 Docker 使用 Google 的私有容器注册表

Google 的 Container Registry 为存储 Docker 图像提供了一个托管的私有存储库。只需一个简单的 gcloud 命令，你就可以推送和拉取你的私有 google 项目库。

示例:

```
gcloud docker — push [HOSTNAME]/[YOUR-PROJECT-ID]/[IMAGE]
```

但是，您可能会发现需要在没有 gcloud 的情况下使用本机 docker 命令。这在 CI 流程或其他自动化中可能是需要的。

示例:

```
docker push [HOSTNAME]/[YOUR-PROJECT-ID]/[IMAGE]
```

在这个简短的教程中，我将通过几个简单的步骤来允许通过本地 docker 命令进行访问。

# 创建一个帐户来访问注册表

设置一些变量来简化命令

```
export PROJECT=my-project
export KEY_NAME=key-name
export KEY_DISPLAY_NAME=”My Key Name”
```

创建并获取密钥

```
gcloud iam service-accounts create ${KEY_NAME} — display-name ${KEY_DISPLAY_NAME}
gcloud iam service-accounts list
gcloud iam service-accounts keys create — iam-account ${KEY_NAME}@${PROJECT}.iam.gserviceaccount.com key.json
```

> 注意:上一个命令的输出是一个名为 key.json 的 json 文件。这个文件将被用作 docker 登录命令的输入，应该被移动到任何需要的系统或位置。

为其提供适当的权限

```
gcloud projects add-iam-policy-binding ${PROJECT} — member serviceAccount:${KEY_NAME}@${PROJECT}.iam.gserviceaccount.com — role roles/storage.admin
```

# 使用凭据访问注册表

登录

```
docker login -u _json_key -p “$(cat key.json)” [https://gcr.io](https://gcr.io)
```

提升你的形象

```
docker push gcr.io/${PROJECT}/example-image
```

也就是说，对于服务帐户 json 文件，您只需调用 login，就可以在您的 CI 或自动化工作中使用 docker