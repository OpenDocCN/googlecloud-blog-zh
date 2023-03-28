# 向 GAE 部署战舰

> 原文：<https://medium.com/google-cloud/deploying-battleship-to-gae-e6aeabf4f34?source=collection_archive---------2----------------------->

这是[战舰](http://www.thagomizer.com/blog/series/battleship.html)系列的一部分。

上次写完战舰服务器。下一步是将其部署到某个地方。我接近了我的第一个真正的截止日期，下周二晚上，所以我采取了简单的方法，将它部署在谷歌应用引擎上。将来，我计划写一篇关于 Kubernetes 部署的博文。我还希望最终将逻辑移植到 Node.js 和 Python，因为我认为语言比较会很有趣。

# 准备工作

我已经有了一个 GCP 账户，并且在我的电脑上安装了 GCP SDK。如果你在家学习，在 App Engine 教程上的 [Rails 5 的“开始之前”框中有这些步骤的说明。自从我上次谈到应用引擎，Ruby 软件工程团队已经发布了一个](https://cloud.google.com/ruby/rails/appengine)[应用引擎 Gem](https://github.com/GoogleCloudPlatform/appengine-ruby) 。这给了我一些在生产中运行 rake 任务的便利特性，并自动引入了 GCP 的 Ruby 监控工具。要使用它，我必须将`gem "appengine"`添加到我的 Gemfile，并将`require "appengine"`添加到我的 Rakefile。

正如我在上一篇文章中提到的，我用的是 Sinatra。我喜欢它的轻量级，但这意味着我必须做一些配置工作。要在 App Engine 上运行应用程序，我需要一个`config.ru`文件。我从 Sinatra 文档中取出一个通用的，并把它放在我的应用程序的根目录下。

```
require './server' 
run Sinatra::Application
```

# 数据库ˌ资料库

我对数据库使用带有 ActiveRecord 的 PostgreSQL。考虑到即将到来的截止日期，我正在使用来自 [CloudSQL](https://cloud.google.com/sql/docs/postgres/) 的托管 PostgreSQL。这样，我就不必担心防火墙规则以及我的 web 服务器和数据库之间的联网。

我按照云 SQL 文档[中的说明](https://cloud.google.com/sql/docs/postgres/quickstart)创建了一个云 SQL。然后，我为我的应用程序创建了一个用户。我把它叫做 rails，因为我总是把我的用户叫做 rails。当我开始写这篇文章时，我才意识到这是不合适的(我用的是 Sinatra)。我还按照[使用 IP 地址连接](https://cloud.google.com/sql/docs/postgres/connect-admin-ip)下的说明创建了一个静态 IP。我将以下内容添加到我的 database.yml 文件中:

```
production:
  <<: *default
  database: "battleship_production"
  username: "rails"
  password: [PASSWORD GOES HERE]
  host:   [YOUR DATABASE IP HERE]
  timeout: 5000
```

使用静态 IP 不是最好的解决方案。有很多安全隐患。我应该使用 [CloudSQL 代理](https://cloud.google.com/sql/docs/postgres/connect-admin-proxy)，但是我想要一些能够快速证明我的部署的东西。我可以稍后返回并更改应用程序连接到数据库的方式。

要使用 appengine gem 运行迁移，我需要将以下内容添加到我的 Rakefile 中。

```
require "appengine"
require "appengine/tasks"
```

# 部署应用程序

将应用程序部署到 Ruby 的 App Engine 很简单，只需输入`gcloud app deploy`。gcloud SDK 将在第一次运行时分析您的应用并提出一些建议。默认是好的。它将使用默认值编写一个类似于下面的`app.yaml`文件:

```
entrypoint: bundle exec rackup -p $PORT
env: flex
runtime: ruby
```

实际部署可能需要一段时间。对我来说，部署大约需要 7 分钟。我希望他们能更快，我知道团队正在努力，但这是在决定在哪里运行你的应用程序时要记住的事情。

部署应用程序后，您可以像这样运行迁移:

```
bundle exec rake appengine:exec -- bundle exec rake db:migrate
```

默认情况下，`gcloud app deploy`会在应用程序启动并运行后立即将所有流量路由到最新版本的应用程序。如果您需要在该版本能够处理流量之前运行迁移，您可以使用`--no-promote`标志:

```
gcloud app deploy --no-promote
```

一旦运行了迁移，您就可以使用 [web UI](https://console.cloud.google.com/appengine/versions) 将流量路由到新版本。

我的应用引擎部署在 https://battleship-176302.appspot.com/[如果你想自己尝试一下的话。预计在接下来的几周内会有一些不稳定，我会整理出一些细节，让它可以用于下周在西雅图举行的研讨会。](https://battleship-176302.appspot.com/)

08/31/17

*原载于 2017 年 8 月 31 日 www.thagomizer.com*[](http://www.thagomizer.com/blog/2017/08/31/deploying-battleship-to-gae.html)**。**