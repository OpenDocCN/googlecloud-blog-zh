# 在谷歌应用引擎上部署流星应用

> 原文：<https://medium.com/google-cloud/deploying-a-meteor-app-on-google-app-engine-97d340c63576?source=collection_archive---------0----------------------->

虽然 Meteor 最近停止了他们的免费托管服务[ [1](https://forums.meteor.com/t/meteor-com-free-hosting-ends-march-25-2016/19308) ]，但谷歌最近测试版发布了支持 Node.js [ [2](https://cloudplatform.googleblog.com/2016/03/Node.js-on-Google-App-Engine-goes-beta.html) ]的应用引擎。这篇短文指导您如何在 Google App Engine 上部署一个 Meteor 应用程序。

## 准备

首先，你需要设置谷歌云平台。请看[的快速入门](https://cloud.google.com/nodejs/getting-started/hello-world)。简而言之，您创建并启用一个云控制台项目，然后下载 Google Cloud SDK 。按照说明做应该很容易。此外，如果您是第一次使用，需要在[控制台](https://console.developers.google.com/apis/api/appengine/overview)激活 API 管理器。

其次，配置权限很重要。一种简单的方法是通过以下方式以所有者身份登录:

```
gcloud login
```

或者，您可以下载具有适当权限的服务帐户的 key.json 并激活它:

```
gcloud auth activate-service-account --key-file key.json
```

最后，初始化 gcloud:

```
gcloud init
```

## 建设

在您的 Meteor 项目目录中，运行以下命令来构建您的应用程序:

```
meteor build .deploy --directory
```

然后，在中创建“package.json”。部署/捆绑以下内容:

```
{
  "private": true,
  "scripts": {
    "start": "node main.js",
    "install": "(cd programs/server && npm install)"
  },
  "engines": {
    "node": "0.10.43"
  }
}
```

您还需要在同一个目录中创建包含以下内容的“app.yaml ”:

```
runtime: nodejs
env: flex
threadsafe: true
automatic_scaling:
  max_num_instances: 1
env_variables:
  MONGO_URL: 'mongodb://[user]:[pass]@[host]:[port]/[db]'
  ROOT_URL: 'https://...'
  METEOR_SETTINGS: '{}'
```

## 部署

英寸部署/捆绑，运行以下命令来部署您的应用程序:

```
gcloud app deploy
```

就是这样！这需要一段时间，但当它完成后，你就万事俱备了。