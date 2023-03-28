# 在 Google Cloud 上部署功能(无服务器)应用

> 原文：<https://medium.com/google-cloud/deploying-a-functions-serverless-app-on-google-cloud-e8f0e3011b2b?source=collection_archive---------2----------------------->

![](img/18295f16f94820b3a20a345ab4286acb.png)

谷歌云

*本教程将向您展示如何将功能应用部署到谷歌云*

# 先决条件

*   您已经安装了`gcloud`工具
*   你在 GCP 有一个项目

> 注意:您必须安装`gcloud`测试版

要安装`gcloud`测试版使用:

```
gcloud components update && gcloud components install beta
```

# 我们开始吧

首先，让我们制作一个目录并输入它:

```
mkdir ~/hello_functions && cd hello_functions
```

现在，制作一个`index.js`文件，并将它放入:

```
exports.helloFunctions = (req, res) => {
  res.send('Hello Functions!');
};
```

这就产生了一个名为`helloFunctions`的函数，它用`Hello Functions!`来响应 GET 请求

# 就这样？

是的，差不多。剩下的就让谷歌云来做吧。

确保您的项目 ID(来自 GCP)被设置为环境变量，并配置`gcloud`来使用它:

```
PROJECT_ID=<project-id>
gcloud config set project $PROJECT_ID
```

现在部署您的函数应用程序:

```
gcloud beta functions deploy helloFunctions --trigger-http
```

> 注意:它可能会要求您启用函数 API，请说“是”

部署完成后，您可以通过以下内容找到您的函数的 URL:

```
gcloud beta functions describe helloFunctions | grep url
```

该 URL 应该类似于:

```
https://[GCP-REGION]-[PROJECT_ID].cloudfunctions.net/helloFunctions
```

在浏览器中打开该 URL，您应该会看到`Hello Functions!`

# 本地测试

如果你想在部署之前在本地测试你的功能，[云功能模拟器](https://github.com/GoogleCloudPlatform/cloud-functions-emulator)让它变得超级简单！

首先安装函数模拟器:

`npm install -g @google-cloud/functions-emulator`

然后启动它:

`functions-emulator start`

现在在本地部署该功能:

`functions-emulator deploy helloFunctions --trigger-http`

并测试它！

`functions-emulator call helloFunctions`

您应该会看到类似这样的内容:

```
ExecutionId: d02ce9e7-bf2c-44fb-89ab-ab8445277ecd
Result: Hello Functions!
```

# ⒡ → 👋功能

感谢阅读！如有疑问，在下方评论！