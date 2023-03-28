# 我对谷歌云端点的看法{GO}

> 原文：<https://medium.com/google-cloud/my-thoughts-on-google-cloud-endpoints-go-cede18edd66f?source=collection_archive---------1----------------------->

## 序言

几个月前，我与@ [阿利克](https://goo.gl/ETmZkI)和@ [沙哈尔](https://goo.gl/WgZRDM)坐在一起，讨论为[垃圾箱应用](http://dumpsterapp.mobi/)构建一个 Rest API，以便将用户文件提供到云端，而不是他们的手机设备上。我们争论应该用哪种语言开发 rest API。对于我这个熟悉 Scala 的 Java 开发人员来说，我认为 Scala 和 [spray.io](http://spray.io/) 是最好的选择，如果真的有人决定使用 Scala，这可能是真的。但是 Alik 认为 GO 是一种新的“东西”,被认为非常容易学习，并且对于构建 rest API 来说非常快。(相对于学习 scala 来说确实如此。)

## 我应该使用哪个 GO Rest API 框架？我应该使用哪个 GO Rest API 框架？

有很多 rest api 框架，其中一定有基于 net/http [框架](https://github.com/julienschmidt/go-http-routing-benchmark)的

作为一个围棋新手，我有点不知所措……
我玩了玩 [Begoo](http://beego.me/) (非常好，但是比我们需要的功能更多)。

然后，我尝试了大猩猩工具包，它非常简单明了，我相信我会在未来的某个时候使用它。
我们尝试云端点的主要原因是 Android 代码生成(Jars ),但是我们在里面发现了更多好东西。

## 云端点是好东西

**API 测试工具**

API explorer 是一个很好的工具来测试你的应用程序，糟糕的是你不能保存你以前的测试和运行自定义测试。

**剩余文件生成器**

https://{ appname } appspot . com/_ ah/API/discovery/{ version }/API/{ API name }/{ version }/rest

如果你输入这个 url 到你的浏览器，你会得到一个很好的 rest API 文档。我们是一个小团队，但是在一个更大的项目中，它会非常有用。

**json 编码/解码**

在 GO 中实现 json 编码/解码并不复杂，但是在端点中，您可以开箱即用。

**代码生成**

你可以以多种方式使用 rest API，但使用云端点非常简单，android 团队对此非常满意，我们可能会生成 Ios 或 Javascript 版本。

**云端点坏东西**

恭喜你嫁给了谷歌云…

如果你想在其他地方运行你的代码，你必须改变很多东西。

## 收场白

Google end points 是在 app engine 上创建 Rest API 的非常有效的工具，如果你打算使用 Google 云平台，你应该使用它。