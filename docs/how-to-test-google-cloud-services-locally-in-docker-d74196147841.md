# 如何在 Docker 本地测试 Google 云服务

> 原文：<https://medium.com/google-cloud/how-to-test-google-cloud-services-locally-in-docker-d74196147841?source=collection_archive---------0----------------------->

“gcloud”命令行工具令人惊叹，它超级简单，工作起来没有任何问题，您可以使用它进行身份验证，它会打开一个浏览器窗口，然后设置正确的值，这比您必须在 bash 配置中手动设置的 access 和 secret 令牌要容易得多。

然而随之而来的问题是，如何在本地运行的 docker 容器中获得授权。

尤其是现在“gcloud auth login”说:

```
WARNING: `gcloud auth login` no longer writes application default credentials.
```

# 解决办法

我们需要运行这个命令，将认证信息写到一个文件中

```
$ gcloud auth application-default login
```

这会将它保存到以下文件中:

```
Credentials saved to file: [/Users/kevinsimper/.config/gcloud/application_default_credentials.json]
```

现在，您需要将 docker-compose.yml 文件更改为:

```
volumes:
  - ~/.config/:/root/.config
```

这将使`google-cloud`包寻找正确的认证位置。

成功，可以使用 docker 内部的 gcloud 组件。