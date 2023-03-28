# 将 Firebase 云功能迁移到节点 8

> 原文：<https://medium.com/google-cloud/migrating-firebase-cloud-functions-to-node-8-aebdb0d3d9a9?source=collection_archive---------0----------------------->

这篇文章的目的不是给出全面的指导，而是指出从节点 6 迁移到节点 8 时的一些潜在问题。如[官方文件](https://firebase.google.com/docs/functions/manage-functions#set_nodejs_version)中所述，要求很简单。

*   确保`firebase-functions`库是 2.0.0+
    `npm install firebase-functions@latest --save`
*   确保 firebase-tools 版本为 4.0.0+
    `npm install -g firebase-tools --upgrade`
*   将`"engines": {"node": "8"}`添加到`functions/`目录下的`package.json`中。

现在就我的经验谈几点看法。当我从我的本地计算机上进行更改和部署时，这工作得很好。但是，我对我们用于 CI/CD 的云构建配置有一些问题。在我们的例子中，这个问题表现为 async/await 用法的语法错误。我想我们要么使用了错误的节点版本，要么使用了错误的 firebase-tools 版本。

本质上，这与我们如何为云构建配置构建`firebase-tools`映像有关。我们的 docker 文件最初看起来像这样:

```
**# Node 6
FROM node:boron**
# install Firebase CLI
RUN npm install -g firebase-toolsENTRYPOINT ["/usr/local/bin/firebase"]
```

这导致 firebase CLI 在运行`firebase deploy`作为云构建步骤时使用 node6。很难马上发现问题，因为构建这个映像是部署过程中很少的一步。我们只是用它来构建 CLI，并把它放到 Google 容器注册表中，然后就把它忘得差不多了。理想的方法是标记 firebase CLI 的不同节点版本，但当时我没有想到。

长话短说，当您将 Firebase 云功能迁移到 Node 8 并且您是云构建用户时，不要忘记使用正确的 Node 版本重新构建 firebase-tools 映像。正确的答案是:

```
**# Node 8
FROM node:carbon**
# install Firebase CLI
RUN npm install -g firebase-toolsENTRYPOINT ["/usr/local/bin/firebase"]
```

如果你在互联网的这一部分结束，希望它能节省你一些时间。

干杯，