# 在 Google 云函数上创建 MongoDB CRUD 后端。

> 原文：<https://medium.com/google-cloud/creating-a-mongodb-crud-backend-on-google-cloud-functions-88bb5c1cef77?source=collection_archive---------1----------------------->

在过去的两周里，我一直在摆弄谷歌的云功能。在看了来自谷歌([https://twitter.com/bretmcg](https://twitter.com/bretmcg))的 Bret McGowen 在 SF 无服务器会议上的精彩介绍和演示后，我决定试一试。

## 那么谷歌云功能到底是什么呢？

Google Cloud Functions 是 Google 提供的一种功能即服务(FaaS ),计费精确到 100 毫秒，运行 Javascript。

## 首先谷歌云功能，Hello World？

当然，这是一个虚拟的例子，但它是有效的！

这是一个基本的 JavaScript 模块。导出一个名为**T3 的函数句柄 T5。这个函数非常类似于 Expressjs([https://expressjs.com/](https://expressjs.com/))处理函数。所以，任何在 Express 上有效的东西，在这里也有效。(GCF 实际上是使用 Expressjs 来支持它们的 HTTP 功能)**

GCF 有一组最低要求。比如:

*   *package.json* 文件来管理依赖项，它将在部署时安装。
*   Node 6.9.1(令人失望的是我们还不能直接使用以后的版本，但是 Babel 可以帮助我们)
*   GCF SDK 已安装并初始化([https://cloud.google.com/sdk/downloads#interactive](https://cloud.google.com/sdk/downloads#interactive))
*   一个谷歌云存储桶上传或编码(从 Git 上传功能也是可能的)

部署脚本从 *package.json* 文件上的“main”prop 获取函数。

```
{
  "name": "gcf-hello",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "author": "",
  "license": "ISC"
}
```

一旦我们覆盖了所有的基础，我们需要创建一个部署桶。一个小命令将帮助我们:

```
$ gsutil mb gs://medium-post-functions
```

然后，只需用一个“简单”的命令部署该功能，比如:

```
$ **gcloud beta functions deploy hello-world --entry-point handler --trigger-http --stage-bucket medium-post-functions --memory=256MB**Copying file:///var/folders/f9/_xttz7rs7t10grd6_q_8s1dr0000gn/T/tmpxV6pSu/fun.zip [Content-Type=application/zip]...
- [1 files][  438.0 B/  438.0 B]
Operation completed over 1 objects/438.0 B.
Deploying function (may take a while - up to 2 minutes)...done.
availableMemoryMb: 256
entryPoint: handler
httpsTrigger:
url: [https://us-central1-revelatio-165320.cloudfunctions.net/hello-world](https://us-central1-revelatio-165320.cloudfunctions.net/hello-world)
latestOperation: operations/cmV2ZWxhdGlvLTE2NTMyMC91cy1jZW50cmFsMS9oZWxsby13b3JsZC9nb0pHUXV2TXFTbw
name: projects/revelatio-165320/locations/us-central1/functions/hello-world
serviceAccount: revelatio-165320@appspot.gserviceaccount.com
sourceArchiveUrl: gs://medium-post-functions/us-central1-hello-world-bsesoemdvxwb.zip
status: READY
timeout: 60s
updateTime: '2017-05-15T16:20:11Z'$
```

我们将获得一个 URL 来通过 HTTP 调用我们的函数。

```
$ curl [https://us-central1-revelatio-165320.cloudfunctions.net/hello-world](https://us-central1-revelatio-165320.cloudfunctions.net/hello-world)
Hello World$
```

## 添加一些依赖项使其有用

首先，我想在更新的 ES2015 JavaScript 上写这个函数。让我们把巴别尔([https://babeljs.io/](https://babeljs.io/))加入进来。

```
$ yarn add --dev babel-cli babel-core babel-plugin-transform-runtime babel-preset-es2015 babel-preset-stage-1$ yarn add babel-runtime
```

还有一个*。babelrc* 文件:

```
{
  "presets": ["es2015", "stage-1"],
  "plugins": ["transform-runtime"]
}
```

在这之后，我们还可以在我们的 *package.json* 文件中添加一个脚本部分来构建我们的代码(另一个用于构建和部署)

```
{
  "name": "gcf-hello",
  "version": "1.0.0",
  "description": "",
  **"main": "lib/index.js",
  "scripts": {
    "build": "rm -rf lib/ && `yarn bin`/babel index.js --out-dir ./lib",
    "deploy": "yarn build && gcloud beta functions deploy hello-world --entry-point handler --trigger-http --stage-bucket medium-post-functions --memory=256MB"
  },**
  "author": "",
  "license": "ISC",
  **"dependencies": {
    "babel-runtime": "^6.23.0"
  },
  "devDependencies": {
    "babel-cli": "^6.24.1",
    "babel-core": "^6.24.1",
    "babel-plugin-transform-runtime": "^6.23.0",
    "babel-preset-es2015": "^6.24.1",
    "babel-preset-stage-1": "^6.24.1"
  }**
}
```

标记为粗体的文本表示对我们的 *package.json* 文件的更改。

所以现在我们可以用所有更好的 ES2015 编写我们的函数了

```
export const *handler* = (req, res) => res.send('Hello World')
```

而现在，只是:

```
$ yarn deploy
```

您可以根据需要多次重新部署。同一个网址总是回来使用我们的 HTTP 功能。

## 让我们制作一个 MongoDB CRUD API 后端

首先我们需要添加 MongoDB([https://www.npmjs.com/package/mongodb](https://www.npmjs.com/package/mongodb))驱动模块。

```
$ yarn add mongodb dotenv
```

我还在这里添加了一个依赖项。

*   dotenv([https://www.npmjs.com/package/dotenv](https://www.npmjs.com/package/dotenv))，允许我们将配置变量加载到 process.env 中

我个人更喜欢使用 mLab.com 的免费托管 mongodb 服务(不需要超过 500Mb)，你可以在任何地方使用你的 mongodb 数据库，当然可以从互联网访问。MongoDB Atlas 是你可以使用的另一个服务。

一个*。env* 文件将帮助我们配置动态属性，包括 MONGODB 连接字符串。这个文件应该不受版本控制。(例如。*。gitignore*

```
MONGODB=mongodb://gcf:KjyCaENsTT6q8PVE@ds143231.mlab.com:43231/gcf-hello
```

再次部署并从控制台测试它

```
$ curl [https://us-central1-revelatio-165320.cloudfunctions.net/hello-world](https://us-central1-revelatio-165320.cloudfunctions.net/hello-world)[{"_id":"591a25ea734d1d1cd0becda2","name":"Ernesto F.","email":"ernestofreyreg@gmail.com"}]
```

现在我们有来自 CRUD 的 R 部分。让我们尝试添加 C 部分(创建)

```
$ yarn deploy
```

还有…

```
$ curl -d '{"name":"Lucian Freyre", "email":"lucian@codexsw.com"}' -H "Content-Type: application/json" -X POST [https://us-central1-revelatio-165320.cloudfunctions.net/hello-world](https://us-central1-revelatio-165320.cloudfunctions.net/hello-world){"result": "ok"}$ curl https://us-central1-revelatio-165320.cloudfunctions.net/hello-world[{"_id":"591a25ea734d1d1cd0becda2","name":"Ernesto F.","email":"ernestofreyreg@gmail.com"},{"_id":"591a2e757b0c5400028ad0c3","name":"Lucian Freyre","email":"lucian@codexsw.com"}]
```

到目前为止，一切顺利，现在我们已经有了端点，2 个基本操作(创建和读取)。我不会在这里添加更多的操作，你可能已经知道如何添加其余的。

## 最后的想法

*   FaaS 会留在这里。
*   谷歌云功能可以改进很多，特别是在工具、文档和性能方面。

## 后续步骤

*   当然是 GraphQL([http://graphql.org/](http://graphql.org/)))考虑到我们实际上使用 Expressjs 作为后端，使用正确的 graph QL 模块，剩下的工作应该相当容易。
*   和服务器端呈现的反应。