# 鉴定谷歌云红宝石宝石

> 原文：<https://medium.com/google-cloud/authenticating-google-cloud-ruby-gems-adc6fc31f63a?source=collection_archive---------0----------------------->

随着我们为谷歌云产品推出更多的 ruby gems，我一直在询问如何认证你的脚本。有几种不同的方法。有些更适合特定的场景。我想我应该写下有哪些选择，以及何时选择使用每种方法。关于认证的更多信息可以在文档中找到[。](https://googlecloudplatform.github.io/google-cloud-ruby/#/docs/google-cloud/v0.23.0/guides/authentication)

下面的说明假设您已经为项目启用了将要使用的 API。如果你没有这样做或者不确定你可以检查一下 [API 管理器](http://console.cloud.google.com/)。

# 在谷歌计算引擎上运行

我将从简单的例子开始。当您使用 Google Cloud Ruby gems on Compute Engine 运行代码时，身份验证会自动进行。您只需要在创建实例时设置正确的范围。如果 GCE 是您的生产环境，这种方法很有效。然而，由于我的大部分开发工作都是在笔记本电脑上完成的，这对于我来说还不够。当我在 GCE 之外运行我的代码时，我需要一种认证的方法。幸运的是，还有其他几种身份验证方法可用。

# 使用服务帐户

要从外部计算引擎访问 Google Cloud APIs，您需要一个服务帐户。服务帐户类似于您可能在其他服务中使用的 API 令牌或密钥。使用令牌的站点通常使用令牌授予对任何代码的完全访问权限。服务客户可以让您更加细化。每个服务帐户都有一个相关的私钥，您可以授予该帐户访问它需要的 API 和产品的权限。如果帐户的私钥被错误的人共享(或意外登录)，你可以在谷歌云控制台中快速撤销访问权限。

您将在 Google Cloud Console 中生成一个新的服务帐户，并将一个 JSON 文件下载到您的客户端，该文件包含库知道如何读取的密钥。创建服务帐户的说明可从这里获得:[文本](https://googlecloudplatform.github.io/google-cloud-ruby/#/docs/google-cloud/v0.23.0/guides/authentication)或[视频](https://www.youtube.com/watch?v=tSnzoW4RlaQ)。

# 使用环境变量

所有库都需要项目 id 和服务帐户信息来进行身份验证。向代码提供这些信息的最简单的方法之一是使用环境变量。所有的`google-cloud-ruby`库都寻找`GOOGLE_CLOUD_PROJECT`环境变量来获取项目 ID，并寻找`GOOGLE_CLOUD_KEYFILE`(JSON 文件的路径)或`GOOGLE_CLOUD_KEYFILE_JSON`(实际的 JSON)来获取服务帐户信息。

大多数 ruby 爱好者都熟悉这种方法。一旦你设置好了，你就不需要在你的代码中考虑认证了；它只是工作。然而，“它只是工作”也是使用这种方法的缺点。如果您有多个授权范围不同的 GCP 项目或服务帐户，那么在项目之间移动时很容易忘记更新环境变量。此外，如果你和我一样，你会花 20 分钟的时间试图找出为什么有些东西不能被授权，然后你才想起有一个环境变量指向一个不同于你当前工作的项目。

# 在代码中指定

我通常通过在代码中指定凭证来进行身份验证。我经常同时有四个或更多的活动项目。我试着使用环境变量，但是一直搞不清楚我在哪个项目中。所以我转而将认证细节放入代码中。

我创建了一个服务帐户，并将 JSON 文件存储在项目的根目录中。我添加了一个`.gitignore`规则，这样我就不会意外地将凭证签入源代码控制。要使用服务帐户，我只需在实例化 gcloud 对象时传递项目 id 和服务帐户的路径。

```
require "google/cloud" 
gcloud = Google::Cloud.new "my-project-id", "service-account.json"
```

# 组合方法

我也看到人们使用多种认证方法的组合。大多数情况下，他们会在生产、暂存和其他共享环境中使用环境变量，并在开发时在代码中使用凭证。

```
require "google/cloud" 
PROJECT_ID = ENV["GOOGLE_CLOUD_PROJECT"] || 
  "my-project-id" 
SERVICE_ACCOUNT = ENV["GOOGLE_CLOUD_KEYFILE"] || 
  "service-account.json" gcloud = Google::Cloud.new PROJECT_ID, SERVICE_ACCOUNT
```

我不太喜欢这种方法。我喜欢让我的生产环境尽可能与开发相匹配。然而，它应该工作得很好，如果这符合你的需要，对你来说有意义，那就去做吧。

这就是使用 Google Cloud gems for Ruby 时几种不同的认证方式。希望这些方法中的一个能够满足您的需求，您可以很快开始编码。

01/08/17

*原载于 2017 年 1 月 11 日*[*www.thagomizer.com*](http://www.thagomizer.com/blog/2017/01/11/authenticating-google-cloud-ruby-gems.html)*。*