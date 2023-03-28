# #1 段代码:ExpressJS + Memcached 会话

> 原文：<https://medium.com/google-cloud/1-piece-of-code-expressjs-memcached-sessions-884d7d0d0c26?source=collection_archive---------0----------------------->

> 我的机器又出问题了！

*   使用 NodeJS + ExpressJS 的 Google 应用程序引擎设置(ok)
*   使用 PassportJS 和 Google 策略进行身份验证(ok)
*   角度登录(生产失败)

嗯，我开始用运行在谷歌云平台([此处](https://cloud.google.com/nodejs/resources/frameworks/express))上的 ExpressJS 开发一个 app，在尝试保护角度路线时遇到了一些麻烦。我的代码只在本地环境下工作。为什么？仅仅因为 ExpressJS 会话链接到一个实例，而我在生产中有两个实例…好吧。但是我该怎么解决呢？使用 **memcached** 进行会话！

根据 Google App Engine 上的示例 express . js+Memcached Sessions([此处](https://github.com/GoogleCloudPlatform/nodejs-docs-samples/tree/master/appengine/express-memcached-session)):

> 每个 Google App Engine 应用程序都带有一个 memcached 服务实例。

所以成功的两个步骤:

*   在 **config.json** : `MEMCACHE_URL: memcache:11211`中添加 memcached url
*   更新 **server.js** :