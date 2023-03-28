# 将 Google Cloud SDK 与 Powershell 结合使用

> 原文：<https://medium.com/google-cloud/using-the-google-cloud-sdk-with-powershell-dacbd3581208?source=collection_archive---------0----------------------->

[Google Cloud SDK](http://cloud.google.com/sdk) 包括一个名为`gcloud`的命令行工具，方便管理 [Google Cloud](http://cloud.google.com/) 资源。例如，我可以用它来观察和修改一个 [App Engine](https://cloud.google.com/appengine/docs/flexible/) 应用。在这里，我列出了一个 App Engine 应用程序的所有版本:

观察我的应用程序的当前状态很好，但是如果我真的想用这些信息做些什么呢？上面的输出看起来像 Powershell 表，但实际上，它只是文本。如何用 Powershell 解析输出？

# `gcloud` + `ConvertFrom-Json`

`gcloud`和 [Powershell](https://docs.microsoft.com/en-us/powershell/scripting/powershell-scripting?view=powershell-6) 的`[ConvertFrom-Json](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/convertfrom-json?view=powershell-6)` cmdlet 是为对方制作的。`gcloud`可以将其所有输出格式化为 json，`ConvertFrom-Json`将 json 转换为原生 Powershell 对象。

在下面的例子中，我让这两个特性为我服务。

# 删除旧的应用引擎版本

我有一个[应用引擎](https://cloud.google.com/appengine/docs/flexible/)应用程序，它的旧版本已经不再运行，如下面`SERVING_STATUS`栏中的`STOPPED`所示。我想删除这些旧版本。

首先，我添加了`--format json`标志来将`gcloud`的输出格式化为 json。然后，我将结果传输到`ConvertFrom-Json`以将输出转换为 Powershell 对象:

然后我添加了一个过滤器来查找被停止的版本:

最后，我用所有停止的版本 id 调用`gcloud app versions delete`来删除它们:

# 结论

`gcloud`的 json 输出和 Powershell 的`ConvertFrom-Json` cmdlet 让操作谷歌云资源的重复性任务自动化变得很容易。现在 [Powershell 运行在 Linux 和 Mac](https://docs.microsoft.com/en-us/powershell/scripting/setup/installing-powershell-core-on-macos-and-linux?view=powershell-6) 上，你也可以在那些平台上使用这些相同的技术！

下面是一个组合语句，它一步就完成了所有工作: