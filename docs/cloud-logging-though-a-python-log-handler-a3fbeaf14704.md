# 通过 Python 日志处理程序进行云日志记录

> 原文：<https://medium.com/google-cloud/cloud-logging-though-a-python-log-handler-a3fbeaf14704?source=collection_archive---------0----------------------->

我在谷歌云平台上做大数据已经有一段时间了，我真的看到这个平台成熟了。如果你在做大数据，你可能会喜欢 Python。真的是一门把事情做好的语言。

其中一个内置特性是 ***日志*** 。Python 有一些现成的日志处理程序，用于记录控制台、文件和流。但是我真的很惊讶没有人为 Google Cloud Logging 编写一个像样的日志处理程序。

所以我决定为我的 [**luigi-gcloud**](https://github.com/alexvanboxel/luigiext-gcloud/wiki/Logging) 扩展编写一个内置处理程序，通过日志配置让用户完全控制他们想要记录的内容。

但是因为它很小而且易于使用，所以我想分享它，这样你就可以在你的 Python 脚本中构建它，这些脚本应该在 Compute 上运行。将它包含在您的项目中，您就可以使用该处理程序了。

剩下的唯一一件事就是用一个 ini 文件来配置它。不过需要注意的是，要确保根日志记录器不使用我们的处理程序。这是因为客户端库也做日志记录，你会得到一个封闭的循环。

ini 文件中的重要部分是**【logger _ app】**和**【handler _ cloudLoggingHandler】**部分。这将把名为“my-app”的记录器路由到相应的处理程序。在 handler 部分，您需要指定我们的云项目的 project_id。

```
**[loggers]** keys = root,app

**[handlers]** keys = consoleHandler,cloudLoggingHandler

**[formatters]** keys = simpleFormatter

**[logger_root]** level = DEBUG
handlers = consoleHandler

**[logger_app]** level = DEBUG
handlers = cloudLoggingHandler,consoleHandler
qualname = my-app
propagate = 0

**[handler_consoleHandler]** class = StreamHandler
level = DEBUG
formatter = simpleFormatter
args = (sys.stdout,)

**[handler_cloudLoggingHandler]** class = gcloud.log.CloudLoggingHandler
level = DEBUG
args = ('your-project-id',)

**[formatter_simpleFormatter]** format = %(levelname)-8s %(name)-18s : %(message)s
```

一切都配置好了，现在可以开始直接登录云日志了。确保查看控制台中的**自定义日志**。

我必须指出，对于每个日志调用，都会执行一个 API 调用，这会影响性能。但是由于许多 Python 脚本是为对性能不太重要的自动化任务编写的，所以这对他们来说应该不是问题。

伐木快乐！