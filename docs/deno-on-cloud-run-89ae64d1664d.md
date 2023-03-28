# 云运行中的 Deno

> 原文：<https://medium.com/google-cloud/deno-on-cloud-run-89ae64d1664d?source=collection_archive---------1----------------------->

![](img/053e5232a289f62b28e4fd7823039fed.png)

Deno +云运行

🦕**Deno**([Deno . land](http://deno.land))是一个基于 V8 和 Rust 构建的安全的 TypeScript 运行时，由 Node.js 的创建者 Ryan Dahl 创建。

🏃**Cloud Run**([Cloud . Run](http://cloud.run))是一个完全托管的计算平台，可以自动扩展您的无状态容器。

在这篇博文中，我们将向您展示如何在 Cloud Run 上封装和部署一个简单的 Deno 应用程序，运行一个无服务器的 HTTPS TypeScript 服务。

# 创建 Deno 应用程序

让我们用 TypeScript 为 Deno 创建一个 web 应用程序:

## 安装 Deno

使用官方网站上的[说明在您的系统上安装 Deno:](https://deno.land/x/install/)

```
curl -fsSL https:*//deno.land/x/install/install.sh | sh*
```

## 创建 main.ts 文件

使用 Deno 的标准库创建一个简单的 HTTP 服务器:

主页面

您会注意到像`import`和顶级`for await`这样的非节点特性。整洁！

> 你可能会注意到一些代码编辑器，比如 VS Code 对 Deno 特性发出警告，比如顶级 await 或 TS imports ( [GitHub issue](https://github.com/Microsoft/TypeScript/issues/27481) )。它们可以被忽略。

## 本地运行

在继续之前，让我们确保我们的服务器在本地运行:

```
deno run --allow-env --allow-net main.ts
```

点击`localhost:8080`看`Hello, Deno!`

## 创建 Dockerfile 文件

让我们将这个应用程序容器化。

通过简单的`Dockerfile`向码头工人说明如何创建我们的形象:

Dockerfile 文件

为了测试这个`Dockerfile`，我们可以用这个命令在本地运行我们的服务:

```
docker build -t app . && docker run -it --init -p 8080:8080 app
```

# 部署到云运行

现在我们已经创建了我们的`main.ts`和`Dockerfile`文件，我们准备好部署到云运行。

构建容器(`hellodeno`)并部署到云运行(完全托管):

```
GCP_PROJECT=$(gcloud config list --format 'value(core.project)' 2>/dev/null)
gcloud builds submit --tag gcr.io/$GCP_PROJECT/hellodeno
gcloud run deploy hellodeno --image gcr.io/$GCP_PROJECT/hellodeno --platform managed --allow-unauthenticated
```

您将从部署的服务中获得一个 URL:

```
[https://deno-ff-q7vieseilq-ue.a.run.app](https://deno-ff-q7vieseilq-ue.a.run.app)
```

Voila! 棒棒的！

> 这个示例应用程序也是 Knative Docs 中的 [Github 示例。](https://github.com/knative/docs/tree/master/community/samples/serving/helloworld-deno)

# 后续步骤

感谢阅读！看看这些相关的帖子:

*   [云上运行的⚡节点 12 功能](/google-cloud/node-12-functions-on-cloud-run-d891dd93c7c8)
*   [🐛VS 代码本地调试节点 Google Cloud 函数！](/google-cloud/debugging-node-google-cloud-functions-locally-in-vs-code-e6b912eb3f84)
*   [🎥视频:Deno 介绍](https://www.youtube.com/watch?v=AoAXcW2-LNA)
*   [🌐Deno 第三方模块(deno.land/x)](http://deno.land/x)