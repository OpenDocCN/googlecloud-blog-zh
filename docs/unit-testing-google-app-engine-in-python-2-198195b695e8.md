# Python 2 中的单元测试+ Google App 引擎

> 原文：<https://medium.com/google-cloud/unit-testing-google-app-engine-in-python-2-198195b695e8?source=collection_archive---------0----------------------->

在我的上一篇文章中，我解释了如何让 GAE 在本地使用你自己的 pip 包，我只需要再写一篇简短的文章，谈谈如果你依赖谷歌应用引擎，例如，如果你有 NDB 模型的 deps，你如何对你的 pip 包进行单元测试。

目前我最喜欢的 IDE 是 PyCharm，所以我的例子发生在一个环境中，我有以下设置:

*   皮查姆
*   谷歌应用引擎 SDK
*   我自己的 pip 包(带有 setup.py 的 Python 包)带有到 GAE 的 deps
*   Python 2.7.10
*   MacOS 10.13.2(高塞拉)

老实说，我不确定其他 python 开发者是否一直在使用 *virtualenv* ,但是我已经知道 GAE·德夫用这个工具工作得更好。然而，当您使用 PyCharm 时，默认设置是使用系统中的默认 Python 解释器，在我的例子中:

```
Python 2.7.10 located at */usr/bin/python*
```

这让事情变得很有趣，因为即使您参照 GAE SDK 设置您的 PyCharm 项目，测试运行程序也不会理解或导入您的常见 GAE 包，例如:

```
google.appengine.ext
google.appengine.api
```

我立刻找到了这个资源:

*   [https://cloud . Google . com/app engine/docs/standard/python/tools/localunittesting](https://cloud.google.com/appengine/docs/standard/python/tools/localunittesting)

但是做:

```
**import** sys
sys.path.insert(1, *'google-cloud-sdk/platform/google_appengine'*)
sys.path.insert(1, *'google-cloud-sdk/platform/google_appengine/lib/yaml/lib'*)
sys.path.insert(1, *'myapp/lib'*)
```

对我没用——所以在谷歌搜索了一下后，我在 Jetbrains 社区论坛上找到了一个[的帖子，说我应该把这个放在我的*‘my _ test . py’*的顶部](https://intellij-support.jetbrains.com/hc/en-us/community/posts/206630245-Unable-to-run-unit-tests-inside-a-Google-app-engine-project)

```
**import** dev_appserver
dev_appserver.fix_sys_path()
```

我这样做了，它成功了。

我还做了一个' *pip 安装。-升级'*并在本地重新安装我的包。

PS！当然，所有这些都意味着您已经安装了 [Google App Engine SDK](https://cloud.google.com/appengine/downloads) 。

对于样品，我想推荐位于以下位置的样品:

*   [https://github . com/Google cloud platform/python-docs-samples/tree/master/app engine/standard/local testing](https://github.com/GoogleCloudPlatform/python-docs-samples/tree/master/appengine/standard/localtesting)