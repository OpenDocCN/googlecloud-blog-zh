# 向 Slack 发送云构建通知

> 原文：<https://medium.com/google-cloud/google-cloud-build-slack-4eb0e6dd0226?source=collection_archive---------0----------------------->

针对 Google Cloud Build 的 Slack 集成，使用 Google Cloud 函数在构建达到特定状态时向 Slack 发布消息。

Github_Repo : [点击 _ 此处](https://github.com/harsh4870/Cloud-build-slack-notification)

# 设置

1.  创建一个 Slack 应用程序，并复制 webhook URL:

```
Add the value in Index.js File SLACK_WEBHOOK_URL=''
```

2.设置一个 github 令牌，以获取 slack 消息中的 github 提交作者信息(如果适用)。

```
Add the value in Index.js File GITHUB_TOKEN=''
```

1.  使用[无服务器框架(选项 1)](https://github.com/harsh4870/Cloud-build-slack-notification/blob/master/README.md#serverless) 创建功能

# 选项 1:使用无服务器框架部署

1.  安装`serverless`

```
npm install serverless -g
```

1.  确保`serverless.yml`中`project.credentials`的值指向具有适当角色的[凭证，Serverless 可以使用这些凭证在您的项目](https://serverless.com/framework/docs/providers/google/guide/credentials#get-credentials--assign-roles)中创建资源。
2.  [展开](https://serverless.com/framework/docs/providers/google/cli-reference/deploy/)

```
serverless deploy
```

# 拆卸

# 如果使用无服务器框架部署

[移除](https://serverless.com/framework/docs/providers/google/cli-reference/remove/)

```
serverless remove
```

# 常见问题解答

# 这要花多少钱？

每次构建调用 3 次函数:

*   当构建排队时
*   当构建开始时
*   当构建达到最终状态时。