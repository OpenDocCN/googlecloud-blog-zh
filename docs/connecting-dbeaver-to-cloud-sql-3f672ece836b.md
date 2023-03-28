# 将 DBeaver 连接到云 SQL

> 原文：<https://medium.com/google-cloud/connecting-dbeaver-to-cloud-sql-3f672ece836b?source=collection_archive---------0----------------------->

我喜欢通过 [DBeaver](https://dbeaver.io/) 管理我的数据和数据库，我似乎并不孤单。所以在这篇简短的博文中，我们将以两种不同的方式将 DBeaver 连接到[Google Cloud SQL](https://cloud.google.com/sql/)(PostgreSQL)。首先，我们将研究使用本地云 SQL 代理的推荐方法，其次，我们将研究如何使用安全连接将 DBeaver 直接连接到云 SQL。

我将假设您已经创建了一个云 SQL 实例，如这里描述的。

如果您想在容器中运行云 SQL 代理，您可以直接跳到该部分