# 云上运行的 Ruby 函数！💎

> 原文：<https://medium.com/google-cloud/ruby-functions-on-cloud-run-db6ea95ad8ac?source=collection_archive---------4----------------------->

![](img/0c41ce949b2217bd458817aeed5176af.png)

Ruby +云运行

介绍 *Ruby 函数框架*。作为一个服务开发者，这个 gem 允许你用一个简单函数轻松地编写 ruby 服务。在这篇博文中，我们将介绍*如何使用 Ruby Functions 框架来封装和*部署*一个 Ruby 方法到 Cloud Run。*

# 设置

确保 Ruby 2.4+安装了`ruby -v`。[🔗](https://www.ruby-lang.org/en/documentation/installation/)

用以下内容创建一个`Gemfile`:

```
source "[https://rubygems.org](https://rubygems.org)"
gem "functions_framework", "~> 0.1"
```

# 创建一个函数

用下面的函数`hello`创建一个名为`app.rb`的文件:

一句“你好，世界！”app。

# 测试您的功能

通过安装依赖项和启动 web 服务器来测试您的功能:

安装依赖项并启动函数框架。

去`localhost:8080`看看你的函数响应！

# 添加 Dockerfile 文件

Ruby Functions Framework repo 包含一个[示例](https://github.com/GoogleCloudPlatform/functions-framework-ruby/blob/master/examples/echo/Dockerfile) `[Dockerfile](https://github.com/GoogleCloudPlatform/functions-framework-ruby/blob/master/examples/echo/Dockerfile)`，我们可以使用/复制它来构建我们的容器。在您的`app.rb`文件旁边创建一个名为`Dockerfile`的新文件:

Dockerfile 文件

# 部署到云运行

要将您的服务部署到 Google Cloud Platform，请运行以下命令来构建您的容器:

使用云构建构建您的容器

然后将您的容器部署到云运行:

部署到云运行

很快，您将获得一个如下所示的实时 URL:

```
[https://helloruby-q7vieseilq-uc.a.run.app/](https://helloruby-q7vieseilq-uc.a.run.app/)
```

瞧！

# 感谢阅读

您可能也会对这些相关的学习资源感兴趣:

*   [💎Ruby Functions 框架的源代码](https://github.com/GoogleCloudPlatform/functions-framework-ruby)
*   [📄谷歌云文档上的 ruby](https://cloud.google.com/ruby)