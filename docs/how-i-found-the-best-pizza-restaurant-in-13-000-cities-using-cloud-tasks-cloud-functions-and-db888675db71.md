# 我是如何找到最好的披萨店的🍕在 13，000 个城市使用云任务、云功能和谷歌地图🗺️

> 原文：<https://medium.com/google-cloud/how-i-found-the-best-pizza-restaurant-in-13-000-cities-using-cloud-tasks-cloud-functions-and-db888675db71?source=collection_archive---------2----------------------->

![](img/69e386621d5c0b14418c3f247b04a1e9.png)

每个别针都标志着那个城市/镇上最好的比萨饼餐馆。是的，甚至那些看起来在水里的(岛屿)。

你有没有想过，在一个新城市，哪里可以找到最好的披萨？谁不喜欢辣香肠披萨？(或者一块罗马番茄无麸香蒜比萨？)

**谷歌云**为*分布式计算*提供了一些令人难以置信的弹性伸缩技术。最棒的是，你不必考虑技术基础设施是如何工作的。

> 关注你的代码，而不是你的基础设施。

# 它始于两个问题

*   如何显示谷歌云任务的规模？
*   我如何通过编程找到世界上最好的比萨饼切片？

在这一节中，我们将编写一个脚本来创建多个针对 Google Cloud 功能的*任务*。为了让事情变得有趣，**我们将使用谷歌地图 API，在全球 *13，000 个城市*搜索最好的披萨。我们会将结果存储在 Firestore 中。为了可视化我们的数据，一个嵌入了 Google 地图的简单网页显示了每家披萨店的 pin。**

## 我们的应用程序如何工作

这是我们应用程序的架构图:

![](img/efbb31a81f9d59eaa5b9385b66742f25.png)

云任务|云功能|地图 APIs |云 Firestore

在左边的**和**中，我们有排队 13，000 个 HTTP 请求的云任务。

> *为什么我们要在这里使用云任务？也许我们想要*限制*对我们 API 的调用，或者能够*暂停对我们 API 的请求*，或者对于*重复请求*不调用 API。在这里，*预装*我们的 13000 个 URL 目标是很方便的。*
> 
> *这个用例非常适合云任务。*

在图的**中心**，我们有一个 Google Cloud 函数(用 Node 编写)。这种云功能是我们实现以下目标的中心切入点:

*   呈现 Google 地图 Web 用户界面的 Web 浏览器请求
*   获取 Google Maps API 的 API 请求将数据存储在 Firestore 中

> 该应用程序由[节点功能框架](https://github.com/GoogleCloudPlatform/functions-framework-nodejs)提供支持，该框架支持谷歌云功能的本地开发。

在**右侧**中，我们使用[地图地理编码 API](https://developers.google.com/maps/documentation/geocoding/intro) 来查找我们查询的纬度/液化天然气，使用[地图地点搜索 API](https://developers.google.com/places/web-service/search) 来查找纬度/液化天然气附近最好的比萨饼餐馆。结果存储在 Firestore 中(在**底部**)。

# 设置我们的应用程序

首先，一些设置。我们的演示展示了一个端到端的**应用*，它从谷歌地图收集数据，存储在 Firestore 中，并使用具有云功能的公共 URL 显示数据。我们需要设置存储和计算。这相当简单，但需要几分钟。*

## *下载应用程序的代码*

*克隆回购【github.com/GoogleCloudPlatform/cloud-tasks-pizza-map :*

*`git clone [git@github.com](mailto:git@github.com):GoogleCloudPlatform/cloud-tasks-pizza-map.git`*

## *启用 Google APIs*

*我们可以使用[这个特殊链接](https://console.cloud.google.com/flows/enableapi?apiid=cloudtasks.googleapis.com,firestore.googleapis.com,places-backend.googleapis.com,static-maps-backend.googleapis.com,geocoding-backend.googleapis.com)来启用我们项目的所有 API:*

*   *`Cloud Tasks API`:具有重复数据删除、播放/暂停、重试等配置的队列*
*   *`Maps Places API`:寻找披萨店*
*   *`Maps Geocoding API`:查找给定“地点 id”的餐馆的 lat/lng*
*   *`Firestore API`:存储数据*

## ***建立数据库***

*[建立一个 Firestore 数据库](https://firebase.google.com/docs/firestore/quickstart#create)，用于存储我们的 Google Maps API 数据，配置如下:*

*   *模式:`Test mode`*
*   *收藏:`tasks-pizza`*

## *创建一个谷歌地图 API 密钥*

*在此处为地图创建 API 密钥:*

*[![](img/fd40eab143637306e6cccacd7936c65c.png)](https://cloud.google.com/maps-platform/#get-started)

点击此按钮创建 API 密钥。([链接](https://developers.google.com/maps/documentation/geocoding/start#auth))* 

*将密钥保存在如下所示的`.env`文件中:*

```
*KEY=AIzaSyDh7gKIvLzFA0q_ICfkO8ryvEMm3Nrde-c*
```

# *本地运行我们的应用程序*

*要在本地测试我们的服务器，请使用上一步中的`KEY`遵循这些说明:*

```
*npm i
KEY=... npm start*
```

*我们将看到 CLI 输出关于我们应用程序的有用信息:*

```
*> tasks-pizza@1.0.0 start /github/GoogleCloudPlatform/cloud-tasks-pizza-map
> FUNCTION_SOURCE=src npx [@google](http://twitter.com/google)-cloud/functions-frameworkServing function...
Function: function
URL: [http://localhost:8080/](http://localhost:8080/)*
```

*在`localhost:8080`，我们将看到我们的应用程序的路线(我在这里做了注释):*

```
*[
  "/tasks/start", // starts creating all Cloud Tasks
  "/tasks/listnames", // lists Tasks names
  "/maps/add", // adds restaurant data to the database
  "/maps/get", // gets restaurant data
  "/maps/listnames", // lists names of cities from a gist
  "/maps/key", // prints the GMP API key for the front-end
  "/target", // our Cloud Tasks target (same as /maps/add)
  "/web" // our web UI for displaying our map
]*
```

*你在本地运行谷歌云功能！太好了！现在我们如何让这些大头针出现在地图上？🤔*

*这里有一些运行演示的操作顺序:*

*   ***通过运行`/tasks/start`加载云任务队列**。这将需要几分钟的时间，因为它会为世界上的每个城市创建大约 13，000 个独特的任务。*
*   *通过转到[https://console.cloud.google.com/cloudtasks](https://console.cloud.google.com/cloudtasks)，确保云任务**队列未挂起**。取消排队将填充我们的 Firestore 数据库。*
*   ***在`localhost:8080/web`打开我们的前端**来可视化我们数据库中的数据。*

## *测试我们的应用*

*在`/web`，当前端 UI 读取我们的数据库记录时，我们将看到引脚下降。*

*![](img/f39503e427a49fe5008e5ff255d67b31.png)*

*当我们的前端加载我们的结果时，引脚掉落。每根针都是真实的🍕餐厅。*

> *注意:我们故意降低前端速度来制作这个动画，它需要 20 分钟来加载所有的 13k 管脚。*

*例如，在曼哈顿中城，我们有一个到乔氏比萨店的链接，大约有 7k 个评分，平均为 4.5 ⭐s.*

*![](img/1e9d869be00faa81f620cc5ca8d9fbe3.png)*

*纽约曼哈顿中城的乔氏披萨店。*

*那些看起来在海洋里的大头针不是错误。世界上有很多吃披萨的岛屿:*

*![](img/3bd3ccf7e4dc5f7df338a68f739d8fc5.png)*

*佛得角(4.3 ⭐s)的 Boa Pizza 披萨店会根据要求送到海滩。也是“对孩子好”。*

*我已经用这个应用程序在谷歌地图上探索了几个小时的比萨饼世界，并且可以在一整篇文章中谈论最独特的比萨饼餐馆。但是让我们回到开发我们的应用程序。*

# *部署我们的应用程序*

*关于我们的演示，很棒的一点是，将我们的应用程序部署到生产环境中不需要特殊的配置。*

*很容易改变我们的演示，使我们不使用`localhost`，而是在谷歌云上使用一个真正的`https` URL。*

*我们可以简单地运行`npm run deploy`，它实际上运行这个命令:*

```
*gcloud functions deploy tasks-pizza
  \ --trigger-http
  \ --runtime=nodejs10
  \ --env-vars-file=.env.yaml*
```

*大约 30 秒后，您将会看到一个如下所示的公共 URL:*

```
*https://us-central1-my-project.cloudfunctions.net/tasks-pizza*
```

> *注意:那不是一个真实的 URL。*

## *感谢阅读*

*如果你喜欢这篇博客，看看这些资源:*

*   *💻来源:github.com/GoogleCloudPlatform/cloud-tasks-pizza-map*
*   *🌎世界城市数据:【simplemaps.com/data/world-cities *
*   *🔧职能框架:【github.com/GoogleCloudPlatform/functions-framework-nodejs *

*多亏了这个应用程序，我才不会挨饿😋🍕*