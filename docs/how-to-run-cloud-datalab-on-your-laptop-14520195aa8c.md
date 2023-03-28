# 如何在笔记本电脑上运行云数据实验室

> 原文：<https://medium.com/google-cloud/how-to-run-cloud-datalab-on-your-laptop-14520195aa8c?source=collection_archive---------2----------------------->

[云数据实验室](https://cloud.google.com/datalab/)是在谷歌云上运行 Jupyter 笔记本的托管方式。但是如果你运行很多托管服务(Cloud ML Engine，Dataflow，BigQuery 等。)，你其实不需要一个强大的 VM——你的笔记本电脑就够了。同时，在您的本地笔记本电脑上有一个 git 存储库，在 PyCharm 中开发 Python 文件，并且仍然使用 Datalab 进行 GCP 集成，这也是很有帮助的。

你能在笔记本电脑上运行数据实验室吗？

![](img/787757496e040fba65a8bb3d6be0e374.png)

这就是即使您没有连接到云，也可以运行 Datalab 的方法

是的。我就是这么做的(注:这完全是非官方的，不支持):

1.  在你的笔记本电脑上安装 [Docker](https://docs.docker.com/install/) 。
2.  在你的笔记本电脑上安装 [gcloud SDK](https://cloud.google.com/sdk/install)
3.  在终端中，运行“gcloud auth login ”,登录到您的项目
4.  在同一个终端中，保存并运行 run this script(改变内容)变量 first！

```
#!/bin/bash
IMAGE=gcr.io/cloud-datalab/datalab:latest
CONTENT=${HOME}/code/training-data-analyst **# your git repo**
PROJECT_ID=$(gcloud config get-value project)if [ "$OSTYPE" == "linux"* ]; then
  PORTMAP="127.0.0.1:8081:8080"
else
  PORTMAP="8081:8080"
fi docker pull $IMAGE
docker run -it \
   -p $PORTMAP \
   -v "$CONTENT:/content" \
   -e "PROJECT_ID=$PROJECT_ID" \
   $IMAGE
```

现在，您可以简单地浏览到 [http://localhost:8080/](http://localhost:8080/) ，从那里打开本地 repo 中的任何笔记本，并使用 Datalab 对其进行操作。