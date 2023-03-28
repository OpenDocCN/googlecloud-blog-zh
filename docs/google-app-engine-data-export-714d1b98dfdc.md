# 谷歌应用引擎:数据导出

> 原文：<https://medium.com/google-cloud/google-app-engine-data-export-714d1b98dfdc?source=collection_archive---------0----------------------->

[谷歌应用引擎(GAE)](https://en.wikipedia.org/wiki/Google_App_Engine) 是一个 [PaaS](https://en.wikipedia.org/wiki/Platform_as_a_service) ，我用它作为一个老项目的后端。GAE 非常易于使用，具有合理的伸缩性，并允许开发人员专注于构建他们的应用程序。

根据[精益创业原则](http://theleanstartup.com/principles)，我想对我的应用程序中存储的数据进行分析。我立即面临两个问题:

1.  GAE 不允许您直接连接到您的(托管)数据库。
2.  [App Engine 使用 Google 查询语言(GQL)，与无处不在的结构化查询语言(SQL](https://en.wikipedia.org/wiki/Google_App_Engine#Differences_between_SQL_and_GQL) )略有不同。

以上两个路障促使我写了一个[Google App Engine 的数据导出工具，我已经在 Github](https://github.com/nikhilsaraf/GAEDataExport) 上开源了。这个工具帮助你使用 Google 提供的 [gsutil](https://cloud.google.com/storage/docs/gsutil) 工具解码你从 App Engine 下载的 db 文件。通过这个工具下载数据是运行 [GAEDataExport](https://github.com/nikhilsaraf/GAEDataExport) 的先决条件(我想我应该给这个工具起个更恰当的名字)。

我的项目的主要目标是将下载的数据库转换成容易阅读和普遍理解的 CSV 文件。一旦你有了 CSV 格式的数据，你就可以把它上传到你选择的任何数据库中，因为大多数数据库都能读取 CSV 格式的文件。我将很快添加一篇关于我的另一个 OSS 项目的博文，它允许你将这些(或任何其他)CSV 文件导入到你的 Postgres 数据库中。

我一直定期使用这个 GAEDataExport 工具来导出这样的数据，到目前为止，它对我来说工作得非常好。我确实有一些[未解决的问题](https://github.com/nikhilsaraf/GAEDataExport/issues)需要解决，如果你有兴趣参与进来，我将非常感谢你的帮助！如果没有，那么我希望你能像我一样找到有用的工具！

同-EN