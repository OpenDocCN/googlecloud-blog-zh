# 带有云 SQL 代理的应用引擎上的 Node.js

> 原文：<https://medium.com/google-cloud/node-js-on-app-engine-with-cloud-sql-proxy-43f56279a56b?source=collection_archive---------0----------------------->

过去几周，我一直在玩谷歌云，试图建立一个节点项目。许多服务目前都处于测试阶段，需要反复试验才能理解大部分服务是如何工作的，最终会变得相当简单。

主要组件:

*   云 SQL 第二代
*   云 sql 代理(让我们不必打开所有 IP 的访问)
*   部署在应用程序引擎上的 Node.js express 应用程序
*   ORM 的序列

大部分信息都在实际的文档中，但是这里有一些对我有用的代码。

**用于开发:**

一旦安装到您的机器上，运行代理进程(在 OSX 上)并保持运行，以便连接通过。

> /cloud _ SQL _ proxy-dir =/cloud SQL-instances = { project name }:{ zone }:{ instance-name } = TCP:3306

在我的例子中，{ project name }:{ zone }:{ instance-name }类似于 proj1:us-central1:dev-db

本地连接配置为:

> “用户名”:“dev”，
> “密码”:“devpwd”，
> “数据库”:“dev-db”，
> “主机”:“127.0.0.1”，
> “方言”:“mysql”

**用于生产:**

对于生产来说稍微有点复杂，但关键是设置 mysql 连接以使用套接字:

> " username": "dev "、
> "password": "devpwd "、
> "database": "dev-db "、
> "host": "localhost "、
> "dialect": "mysql "、
> " dialect options ":{
> " socket path ":"/cloud SQL/{ project name }:{ zone }:{ instance-name } "
> }

最后一部分在 app.yaml 中，用于在正在部署的实例上启用云代理:

> beta _ settings:
> cloud _ SQL _ instances:{项目名称}:{区域}:{实例名称}

希望这可能是一个有益的参考！

乔治

参考资料:

[](https://cloud.google.com/sql/docs/sql-proxy) [## 云 SQL 代理配置

### 这是云 SQL 的测试版。此功能可能会以向后不兼容的方式更改，并且不是…

cloud.google.com](https://cloud.google.com/sql/docs/sql-proxy)