# App Engine 项目清理

> 原文：<https://medium.com/google-cloud/app-engine-project-cleanup-9647296e796a?source=collection_archive---------0----------------------->

[谷歌云平台](https://cloud.google.com/) (GCP)项目和[应用引擎](https://cloud.google.com/appengine/)服务和版本很容易创建，并且可能会激增。随着时间的推移，一些应用引擎服务和版本可能会依赖于云服务，这些云服务已经被[弃用](https://cloud.google.com/appengine/docs/deprecations/)。当废弃的服务最终关闭时，删除旧版本以防止混乱可能是一个好的做法。这篇文章将描述一些使用 [gcloud](https://cloud.google.com/sdk/gcloud/reference/) 命令行工具清理项目的方法。

目标受众是开发人员，他们希望在最终删除旧版本之前，通过一些人工审查来有效地管理相对较少的版本。一个重要的例子是不赞成使用的托管虚拟机(vm:true)运行时。在[升级到 App Engine 灵活环境最新版本](https://cloud.google.com/appengine/docs/flexible/python/upgrading)中讨论了向 Flex 的迁移。请注意，迁移到 Flex 需要一些代码更改。如果您不想进行任何代码更改，那么您可以考虑恢复到 App Engine Standard，而不是迁移到 Flex。

当托管虚拟机首次发布时，它是第一个支持 Nodejs 的应用引擎运行时。所以，这篇文章中的例子使用了 Nodejs。然而，这些想法和命令适用于所有运行时。

**先决条件**
你需要一个应用引擎应用程序的 GCP 项目。

使用 [gcloud config set](https://cloud.google.com/sdk/gcloud/reference/config/set) 命令在本地设置中设置默认项目

```
PROJECT=[Your project]
gcloud config set project $PROJECT
```

如果您还没有一个应用程序，或者想要创建一个测试应用程序，您可以
使用下面的命令创建一个，使用来自应用程序引擎标准环境中的 [Quickstart for Node.js 的代码:](https://cloud.google.com/nodejs/getting-started/hello-world)

```
git clone [https://github.com/GoogleCloudPlatform/nodejs-docs-samples](https://github.com/GoogleCloudPlatform/nodejs-docs-samples)
cd nodejs-docs-samples/appengine/hello-world/standard
gcloud app deploy
```

这将部署一个 Nodejs 应用程序。测试您的应用程序是否可访问。

```
gcloud app browse
```

请注意， [gcloud app deploy](https://cloud.google.com/sdk/gcloud/reference/app/deploy) 命令现在已经取代了之前的
[appcfg](https://cloud.google.com/appengine/docs/standard/python/tools/appcfg-arguments) 命令。

如果我们对包含代码的 app.js 文件进行更改，然后使用基本的 gcloud app deploy 命令再次部署
应用，那么将会自动创建第二个版本。

例如，改变线路

```
.send(‘Hello, world!’)
```

到

```
.send(‘Hello, world 2!’)
```

和执行

```
gcloud app deploy
```

会自动创建新版本的应用程序。旧版本将被保留，但不再接收流量。这在短期内很方便，但会导致许多版本的积累。

要检查应用程序提供的默认页面，请执行以下命令

```
curl $PROJECT.appspot.com
```

**发现服务和版本**T3[g cloud app](https://cloud.google.com/sdk/gcloud/reference/app/)命令可以用来获得 app 的概况:

```
gcloud app describe
```

要获取服务列表:

```
gcloud app services list
```

描述一项服务

```
gcloud app services describe default
```

[gcloud app 版本列表](https://cloud.google.com/sdk/gcloud/reference/app/versions/list)命令给出了流量分流百分比和服务状态。

```
gcloud app versions list
SERVICE VERSION TRAFFIC_SPLIT LAST_DEPLOYED SERVING_STATUS
default 20181119t152033 0.00 2018–11–19T15:22:04–08:00 SERVING
default 20181119t154106 1.00 2018–11–19T15:42:05–08:00 SERVING
```

[g cloud app versions describe](https://cloud.google.com/sdk/gcloud/reference/app/versions/describe)命令可用于识别应用运行时:

```
gcloud app versions describe --service default 20181010t161719
…
env: standard
…
servingStatus: SERVING
…
```

**检查流量**
您可以通过以下命令读取日志来检查您的应用程序是否正在接收流量

```
gcloud app logs read
```

**清理**
删除不提供任何流量的早期版本应用执行

```
VERSION=20181119t152033
gcloud app versions delete $VERSION
```

检查版本是否已被删除

```
gcloud app versions list
SERVICE VERSION TRAFFIC_SPLIT LAST_DEPLOYED SERVING_STATUS
default 20181119t154106 1.00 2018–11–19T15:42:05–08:00 SERVING
```

**迁移**
假设我们有一些流量，想迁移到 Flex。但是，我们希望在删除旧版本之前检查一切是否正常。让我们用 Quickstart 应用程序的 Flex 迁移对此进行测试。编辑 app.yaml 文件，用以下行替换 App Engine 标准环境中 Node.js 的[快速入门的运行时行](https://cloud.google.com/nodejs/getting-started/hello-world)

```
runtime: nodejs
env: flex
```

然后执行以下命令来部署新版本，但不向其发送流量

```
VERSION=flexversion
gcloud app deploy --no-promote --version=$VERSION --quiet
```

注意，这个命令的变体使用版本标志给
版本命名。它还使用 quiet 标志来避免 gcloud 询问需要人工响应的问题，这有利于自动化。

检查版本是否已部署，但未接收流量

```
gcloud app versions list
SERVICE VERSION TRAFFIC_SPLIT LAST_DEPLOYED SERVING_STATUS
default 20181119t154106 1.00 2018–11–19T15:42:05–08:00 SERVING
default flexversion 0.00 2018–11–20T08:56:26–08:00 SERVING
```

请注意，新版本的流量分割为 0.00，因此它提供服务，但没有流量发送到它。我们可以通过在 URL 中包含版本名称来测试特定的版本。举个例子，

```
VERSION=flexversion
curl $VERSION-dot-$PROJECT.appspot.com
```

你可能想进行更广泛的测试。您可以使用命令浏览到特定版本的应用程序

```
gcloud app browse --version=$VERSION
```

如果您正在升级应用引擎标准版本，那么您可以使用以下命令将
流量迁移到该版本

```
gcloud app versions migrate $VERSION --quiet
```

但是，对于 Flex，我们需要使用 gcloud app deploy 命令，现在升级版本。

```
gcloud app deploy --version=$VERSION --quiet
```

可以多次重新部署同一个版本。检查它现在是否是默认版本

```
gcloud app versions list
SERVICE VERSION TRAFFIC_SPLIT LAST_DEPLOYED SERVING_STATUS
default 20181119t154106 0.00 2018–11–19T15:42:05–08:00 SERVING
default flexversion 1.00 2018–11–20T09:41:33–08:00 SERVING
```

请注意，新版本的流量分割现在为 1.00。然后删除旧版本

```
VERSION=20181119t154106
gcloud app versions delete $VERSION --quiet
```

**暂时关闭一个应用**
假设你不确定用户是否真的在使用你的应用。也许有一些流量的日志记录，但不清楚这是否是真正的使用。你
可以用 [gcloud app versions stop](https://cloud.google.com/sdk/gcloud/reference/app/versions/stop) 命令停止你的应用

```
VERSION=flexversion
gcloud app versions stop $VERSION --quiet
```

检查它是否已停止

```
gcloud app versions list
SERVICE VERSION TRAFFIC_SPLIT LAST_DEPLOYED SERVING_STATUS
default flexversion 1.00 2018–11–20T09:41:33–08:00 STOPPED
```

如果需要，您可以再次启动

```
gcloud app versions start $VERSION — quiet
```

**更多自动化**
如果你有许多旧的应用程序版本或服务需要迁移和/或清理，你可能希望实现自动化。您应该考虑使用 Google App Engine 管理 API，如[将您的应用版本部署到应用引擎](https://cloud.google.com/appengine/docs/admin-api/deploying-apps)中所述。