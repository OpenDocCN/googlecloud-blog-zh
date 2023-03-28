# 云功能—创建、测试、部署

> 原文：<https://medium.com/google-cloud/cloud-functions-create-test-deploy-dc2e725778b9?source=collection_archive---------0----------------------->

如何设置、测试和持续部署 Firebase 云功能 1.0 版

> 所有代码展示都可以在[这个](https://github.com/jeremylorino/firebase-functions-unittest-cd) git 仓库中获得。一如既往地随意贡献、偷窃或什么都不做；你的选择；)

所以 Firebase 最近[发布了他们](https://firebase.googleblog.com/2018/04/launching-cloud-functions-for-firebase-1-0.html)[云功能](https://firebase.google.com/docs/functions/beta-v1-diff)的版本 1 。这带来了一些突破性的变化；对于那些在 v1 之前使用 Firebase 云功能的人来说。

它还附带了一个不错的小[单元测试套件](https://firebase.google.com/docs/functions/unit-testing)；哪个*完全*胜过从[Google Cloud](https://medium.com/u/4f3f4ee0f977?source=post_page-----dc2e725778b9--------------------------------)[GitHub](https://github.com/googleapis/nodejs-firestore/tree/master/test)复制粘贴单元测试模拟；)所以让我们开始吧…

# 什么是现成的…

*   两个— [Firebase 身份验证触发器](https://firebase.google.com/docs/functions/auth-events)
*   一— [云发布/订阅触发器](https://firebase.google.com/docs/functions/pubsub-events)
*   五— [Firebase 云功能单元测试](https://firebase.google.com/docs/functions/unit-testing)
*   一— [谷歌云自动构建触发器](https://cloud.google.com/container-builder/docs/running-builds/automate-builds)

# 设置

> [Firebase 建议](https://firebase.google.com/docs/functions/get-started?#set_up_and_initialize)你用 Node.js v.6.11.5 进行本地开发，但是因为我们要用 TypeScript 写所有东西，让 transpiler 变魔术，所以我们将使用 Node.js LTS v8.10.0。它工作正常…不用担心:)

## 克隆 git repo:

```
git clone git@github.com:jeremylorino/firebase-functions-unittest-cd.git
```

## 安装 Node.js 依赖项

```
npm install
```

## 获取一些谷歌云服务帐户凭证

## 使用 [Firebase CLI](https://firebase.google.com/docs/cli/) 进行认证

## 初始化项目目录

> 请确保从列表中选择您的 Firebase 项目

*   从语言提示中选择 TypeScript
*   对 TSLint 是
*   **否**所有的“覆盖”问题
*   是安装依赖项

## 那就测试一下！

```
npm test
```

# 我们刚才做了什么？

让我们深入研究其中一个函数定义…

这个小片段是两个 Firebase 身份验证触发器之一。当创建一个新的 Firebase 用户时，它会将用户的信息写入一个 Firestore 文档，同时写入一个包含用户访问控制有效权限的文档。

由创建 Firebase 身份验证用户触发的云功能

然后我们有相关的单元测试。

*   第 3–9 行:模拟 Firebase *用户记录*
*   第 11–12 行:使用模拟*用户记录*执行*用户创建*函数
*   第 17–28 行:断言函数按预期执行

# 现在进行一些连续部署

首先，我们需要一个 Firebase CI 令牌

```
npm run firebase:citoken
```

*   导航到[构建触发器](https://console.cloud.google.com/project/_/gcr/triggers)
*   连接到 git repo
*   设置“构建配置”以使用 *cloudbuild.yaml*
*   将 *_FIREBASE_TOKEN* 的值设置为从上述命令接收到的值

# 结束了