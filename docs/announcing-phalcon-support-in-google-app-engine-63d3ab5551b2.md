# 在谷歌应用引擎中宣布支持 Phalcon

> 原文：<https://medium.com/google-cloud/announcing-phalcon-support-in-google-app-engine-63d3ab5551b2?source=collection_archive---------3----------------------->

**更新**:我刚刚发现 Phalcon 最高只支持 PHP 7.0，所以要确保你的 composer.json 文件使用的是支持的 PHP 版本(默认版本是 7.1)。

我已经与我们的主要客户 Phalcon 合作了一段时间，我的主要挫折之一是没有将 Phalcon 框架捆绑在 Google App Engine Flex 环境中，因为这迫使我们构建自己的 docker 映像，这大大增加了我们的部署时间。

由 Google 维护的 php-docker 映像的贡献者最近添加了 Phalcon 扩展的 automagic building(以及其他),并且从今天起已经决定将这些默认安装的包包含到 docker 映像中。

# [如何启用 Phalcon](https://devdemand.co/announcing-phalcon-support-in-google-app-engine/#ref3)

如果这里列出的[中的任何一个包](https://github.com/GoogleCloudPlatform/php-docker/tree/master/deb-package-builder/extensions)是您被迫在自己的 docker 映像中构建的扩展，那么您现在可以完全抛弃一个定制的 docker 映像，并在其中包含一个`php.ini`文件和您的`extension=myextension.so`行，以便在部署您的服务时安装它。

请记住，如果您删除 docker 文件，您需要更改您的`app.yaml`，其中`env: custom`将成为`env: flex`。如果你的 app.yaml 中没有`env:`，很可能你使用的是`vm: true`，这是使用 App Engine 的 flex 环境的过时方法，所以你需要将`vm: true`改为`env: flex`。

> ***注意*** *:我们发现从* `*vm: true*` *更改为* `*env: flex*` *会对路由产生一些意想不到的副作用，所以请确保在上线之前测试该更改。*

如果您需要帮助来设置应用引擎环境，请告诉我们。我们即将被认证为谷歌云合作伙伴，并且很乐意为您的谷歌云解决方案提供帮助。

[1][https://github.com/GoogleCloudPlatform/php-docker/pull/229](https://github.com/GoogleCloudPlatform/php-docker/pull/229)
【2】[https://github . com/Google cloud platform/PHP-docker/releases/tag/2017-03-22-19-13](https://github.com/GoogleCloudPlatform/php-docker/releases/tag/2017-03-22-19-13)
【3】[https://github.com/GoogleCloudPlatform/php-docker/pull/237](https://github.com/GoogleCloudPlatform/php-docker/pull/237)

最初发布于[开发需求](https://devdemand.co/announcing-phalcon-support-in-google-app-engine/)