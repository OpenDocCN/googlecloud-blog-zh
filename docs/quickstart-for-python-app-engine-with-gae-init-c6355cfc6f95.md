# 使用 gae-init 的 Python 应用引擎快速入门

> 原文：<https://medium.com/google-cloud/quickstart-for-python-app-engine-with-gae-init-c6355cfc6f95?source=collection_archive---------0----------------------->

你可能听说过[谷歌应用引擎](https://cloud.google.com/appengine/)，甚至可能已经完成了[快速入门](https://cloud.google.com/appengine/docs/python/quickstart)，但最有可能的是，你从未实现拥有一个有用的最终产品的目标。

启动新应用需要的不仅仅是测试 App Engine 如何工作。仅举几个例子，在现实生活中你通常需要:用户管理、不同的登录选项、RESTful API、管理控制台、最小化、任务队列、错误处理、命令行工具等等。

![](img/ef005363c14866bf39c01fef03d00bc1.png)

[**gae-init**](https://github.com/gae-init/gae-init) 就是考虑到所有这些问题而创建的，它是最新的样板文件，可以在几分钟内启动 App Engine 上的应用程序。通过使用它，你将不必担心文件夹结构、开发工具、第三方库和许多其他开发者不喜欢处理的事情。你可以从有趣的部分开始！

在深入项目之前，你实际上希望能够快速地“开发第一部分”。遵循 [**快速入门**](http://docs.gae-init.appspot.com/quickstart/) ，在不到 **10 分钟**(包括部署)的时间内让一切正常运行。

[试一试](http://docs.gae-init.appspot.com/quickstart/)如果您遇到任何问题[报告问题](https://github.com/gae-init/gae-init/issues)，使用文档中的编辑链接或在此留下评论。

该项目在麻省理工学院的许可下是开源的，总是最新的，有一群令人敬畏的贡献者(T21 ),并且在世界各地的使用越来越多。

> 如果你想尝试更贵的东西[，注册并在接下来的 60 天内获得 300 美元](https://cloud.google.com/free-trial/)用于谷歌云平台。

在接下来的帖子中，我将尝试演示如何从头开始创建端到端的 — *相对复杂的* — *应用程序，并且只处理有趣的部分。* ***让编程再次充满乐趣……***