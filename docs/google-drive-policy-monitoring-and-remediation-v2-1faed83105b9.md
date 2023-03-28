# Google Drive 策略警报和补救 v2

> 原文：<https://medium.com/google-cloud/google-drive-policy-monitoring-and-remediation-v2-1faed83105b9?source=collection_archive---------2----------------------->

## 借助 Go 多线程提高速度和规模

这篇文章继承了之前的 [Apps 脚本文章](/@fargyle/google-drive-policy-monitoring-and-enforcement-330989ce0d15)的不足。

概括地说，目的是检测、通知和补救不符合策略的文件夹和文件权限。

*   它沿着[凡赛堤](https://github.com/GoogleCloudPlatform/forseti-security)的路线工作，允许你通过文件夹和域定义想要的共享策略，然后对此进行协调；策略向下继承以匹配驱动器的继承。
*   配置开关允许您删除权限或只是通知。

最初的 Apps 脚本工具有三个主要限制:

*   它将文件报告为不符合策略，这些文件具有[多个祖先](https://www.labnol.org/internet/add-files-multiple-drive-folders/28715/)，并且在其中一个分支上共享是允许的。假阳性。
*   策略是在代码中定义的，这使得它很难维护，特别是对于团队来说。
*   它很慢，所以它超过了 Apps 脚本对大文件夹结构的执行时间限制。

第一个限制是通过仅在所有权限累积后进行验证来解决的，而不是沿着每个分支进行验证。

第二个限制是通过从 Google 电子表格中读取策略来解决的。

Go 解决了第三个限制；与[负载测试实用程序](/google-cloud/load-testing-google-cloud-apis-889e393139ca)类似，Go 例程的多线程能力在这里是一个巨大的优势:它允许代码并行导航文件夹层次结构的所有分支，将执行时间从半小时减少到几十秒。

然而，许多由应用程序脚本抽象的复杂性必须在 Go 中明确解决。本文描述了在这一过程中获得的一些经验教训；这里可以直接去码[。](https://github.com/demoforwork/public/tree/master/DrivePolicyV2)

和往常一样，请测试以确保它的行为符合您的预期；正如您将在下面的驱动部分中读到的那样，这个工具也是如此。

# 谷歌服务

## 批准

Apps 脚本自动发现作用域，并引导您完成适当的授权对话；Go 库抽象了用户交互和授权流，但是您仍然需要自己明确定义范围，并使用适当的凭证调用授权。

Google Cloud 不提供向服务帐户授予 G Suite 作用域(如 Drive)的能力，因此您要么需要拥有该域的管理权限，以向服务帐户授予[域范围的授权](https://developers.google.com/admin-sdk/directory/v1/guides/delegation)，要么通过[快速入门](https://developers.google.com/drive/api/v3/quickstart/go)中所示的 3 脚 OAuth 作为用户进行授权。因为这个工具是针对部门用户的，所以它使用三脚 OAuth。

Apps 脚本还自动提供 Stackdriver 日志记录；你需要在 Go 中自己实现这个。您可以向服务帐户授予 Stackdriver 作用域，如[文档](https://cloud.google.com/logging/docs/setup/go)中所述；但是这需要管理多个凭证:Stackdriver 的服务帐户凭证和 Google Drive API 的 OAuth 凭证。利用 OAuth 流进行 Stackdriver auth，如[裴军·川本的帖子](https://www.jkawamoto.info/blogs/use-access-token-from-google-cloud-go/)中所述，可以让你整合到一个凭证上。您仍然需要项目 id 来授权 Stackdriver 您可以使用 OAuth 包的 tokenFromFile()函数从 OAuth 凭证中提取它，而不是对它进行硬编码；您需要包括一个副本，因为该函数是不可导出的。

因此，授权代码看起来有点像这样:

让我们讨论一下最小特权。该实用程序使用其使用所需的最少特权，即。默认情况下，电子表格和驱动器读取范围以及日志写入范围。只有当您指定标志来修复权限或发送通知时，它才会请求这些操作的作用域。

## 错误处理

正如 Go Blog 的[错误处理部分](https://blog.golang.org/error-handling-and-go)所述，错误类型是一种接口类型，最常用的错误实现是 errors 包的未导出 errorString 类型。

但是，您会注意到，对于 Google API 错误，错误字符串包含一个错误代码。此代码有助于错误处理；您可以解析字符串中的错误代码，但这是不优雅和脆弱的；相反，您希望断言正确的接口类型。这里的关键在错误消息的另一部分:“googleapi: Error”。

您需要导入[google.golang.org/api/googleapi](https://github.com/googleapis/google-api-go-client/blob/master/googleapi/googleapi.go)包并声明适当的类型。您可以检查这个包，或者使用类似 [Examiner](/capital-one-tech/learning-to-use-go-reflection-822a0aed74b7) 的实用程序来发现这个结构(这也是一个很有启发性的反射例子)；然后，您将在错误处理中断言代码类型:

感谢 [RayfenWindspear](https://stackoverflow.com/users/1276480/rayfenwindspear) 对此的协助…

## 驱动 API

各种驱动程序 API 版本在一些有意义的方面有所不同。方法和响应模式(字段标签及其在层次结构中的级别)不同；对于该实用程序来说，最有意义的是，返回的实际权限结果有所不同。

*   v2 API 返回运行该实用工具的用户是其读者的文件的权限信息；只有当用户是编辑者时，v3 API 才返回这些信息。
*   v2 API 总是填充域字段；对于个人用户的共享，这是用户电子邮件地址的域。v3 API 仅填充域共享的域；您需要解析用户的电子邮件地址，以便在共享给个人用户的情况下确定域。

该实用程序使用 v2 API 您可能希望通过注释/取消注释相关的行来更新到 v3 API。这带来了一个有趣的问题:由于 Go 严格的编译时检查，您不能在检查全局变量的 If 语句中封装不同版本的 API 调用。

此外，API 的行为是不断发展的，因此您需要定期根据一组已知的权限对您的实用程序进行回归测试。

## 驱动配额

驱动器配额也是乐趣的来源:Go 例程的多线程吞吐量超过了大文件夹结构的默认驱动器 API 配额。默认的配额是[描述为](https://cloud.google.com/console/apis/api/drive.googleapis.com/quotas)每个用户每 100 秒 1k 次查询，但似乎在更短的时间内实现，因为如果不增加[配额](https://support.google.com/code/contact/drive_quota)，该实用程序将在不到 200 次查询的情况下超过配额；指数补偿只是增加了具有恒定负载的批处理实用程序的负载，因此该实用程序实现了一个命令行等待标志，它使每个 go 例程休眠指定的秒数。

## 电子邮件

我猜这听起来很熟悉；与 Apps 脚本不同，您需要在 Go 中运行自己的脚本。您可以使用 Go 的 [HTML 模板](https://golang.org/pkg/html/template/)获得类似于 Apps 脚本的结果，但仍然需要自己完成大量实际的邮件撰写工作。 [Mohamed Labouardy 的帖子](http://www.blog.labouardy.com/sending-html-email-using-go/)提供了一个很好的介绍。

你需要添加几个额外的部分来[编码复杂的邮件模板](https://stackoverflow.com/questions/37523884/send-email-with-attachment-using-gmail-api-in-golang)并嵌套它们。对于更复杂的需求，你可能想看看像 Jordan Wright 的包。

您的邮件功能将如下所示:

这里您会注意到的一件有趣的事情是[可变函数](https://gobyexample.com/variadic-functions)的调用语法。

## 行程安排

应用程序脚本提供了触发器，使您可以轻松地每天或每周运行您的实用程序。你需要使用 OS 工具，比如 cron 或者部署到 App Engine Standard 并使用它的调度器。

如果您使用的是 cron，那么在运行该工具之前，您需要使用一个 shell 文件来切换到适当的目录；否则，即使您对路径进行了硬编码，它也不会正确读取令牌文件，这与凭证文件不同。

因此，在 crontab -e 中，每天午夜运行 bash 文件，并将控制台输出发送到日志文件，如下所示。

# Golang 特征

## 结构和反射

这个实用程序使用 [mow.cli 命令行包](http://github.com/jawher/mow.cli)，它将标志值分配给字符串指针；为了能够在通知邮件中显示它们，需要一种迭代它们的方法；实现这一点的方法之一是将它们赋给一个[结构](https://gobyexample.com/structs)，然后遍历该结构；事实证明，迭代指针结构的值是非常重要的。[这篇 Stackoverflow 文章](https://stackoverflow.com/questions/23350173/how-do-you-loop-through-the-fields-in-a-golang-struct-to-get-and-set-values-in-a)提供了一个有用的例子([这篇文章](https://stackoverflow.com/questions/18926303/iterate-through-the-fields-of-a-struct-in-go)提供了一个更简单的方法，但是它要求 struct 字段是可导出的)；因为 struct 字段本身就是指针，所以需要额外调用 Elem()。

所以 CLI 和反射代码看起来像这样:

## 映射键存在检查

这个工具使用大量的地图；这意味着要检查一个键是否存在，这在[这个 Stackoverflow 帖子](https://stackoverflow.com/questions/2050391/how-to-check-if-a-map-contains-a-key-in-go)中有描述。

# 后续步骤

该实用程序的良好扩展是:

*   支持基于谷歌组以及域的政策。
*   支持团队驱动。
*   将通知映射推送到表单，这样您就可以每小时扫描一次，每天通知一次。