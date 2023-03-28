# 在 App Engine 上本地开发 PHP 的一个便利技巧

> 原文：<https://medium.com/google-cloud/a-handy-tip-for-local-development-of-php-on-app-engine-4a54b8d8ed61?source=collection_archive---------1----------------------->

在本文中，我将概述这个技巧背后的背景知识，然后告诉你它是什么以及如何使用它。

# 背景故事

我的团队在我们的 PHP 应用程序中广泛使用 Google App Engine，直到最近，我们还在使用 Google 的 [php-docker](https://github.com/GoogleCloudPlatform/php-docker) docker 映像的定制版本(阅读旧版本),我们在生产和本地测试环境中都使用了它。

我们最近的一个需求意味着我必须触摸 php-docker 图像，在触摸图像时，像许多旧代码项目一样，事情开始变得糟糕。图像的过时导致了一系列需要进行排序的故障。

因此，我研究并咨询了我的团队，选择回到 Google 当前的 php-docker 图像，因为我们觉得它有足够的稳定性和功能来取代我们的自定义图像。在做了几次测试后，我发现:

## php-docker 默认图像不能很好地处理卷

原因是，正好在 docker 镜像运行 web 服务器之前运行的`entrypoint.sh`脚本喜欢将所有东西都分配给一些核心权限。这可能意味着，如果您使用的是卷，您在本地将没有权限访问自己的文件，而在我们的应用程序中，我们的缓存文件夹变得不可写。

为了解决这个问题，我[与 GCP 团队](https://github.com/GoogleCloudPlatform/php-docker/pull/390)合作，在 php-docker 映像中引入了一个新特性，如果你正在进行本地开发，这个特性可以让你跳过这个锁定过程。

# 跳过本地锁定

现在跳过本地锁定是一件非常简单的事情。考虑下面的 docker 命令，它将使用您的本地目录作为一个卷和您的`index.php`文件所在的`/app/public`:
`docker run -v /home/daniel/Repositories/my-project:/app -e DOCUMENT_ROOT=/app/public --name my-project gcr.io/google_appengine/php:latest`

要修改这个命令来停止锁定，只需添加一个新的设置为 true 的 env 变量:`SKIP_LOCKDOWN_DOCUMENT_ROOT=true`，就像这样:
`docker run -v /home/daniel/Repositories/my-project:/app -e DOCUMENT_ROOT=/app/public -e SKIP_LOCKDOWN_DOCUMENT_ROOT=true --name my-project gcr.io/google_appengine/php:latest`

就是这样！没有更多的 chmod 时，本地工作，让您在一个应用程序引擎一样的环境中工作！

如果这个技巧对你有帮助，我很乐意收到你的来信。如果您有任何反馈或编辑，请在下面评论。