# 负载测试谷歌云 API

> 原文：<https://medium.com/google-cloud/load-testing-google-cloud-apis-889e393139ca?source=collection_archive---------2----------------------->

## JMeter 和 Go

Google Cloud Platform 提供了在云控制台中[监控配额和请求增加](https://console.cloud.google.com/iam-admin/quotas)的能力。

这提供了极好的透明性和控制，但并不排除负载测试的需要:

*   谷歌云平台的[深度防御](http://services.google.com/fh/files/misc/csuite_security_ebook.pdf)的一个方面是，基础设施的多个层都有自己的防止滥用的保护措施，这可以表现为负载限制。
*   一些谷歌云 API 的配额，如[管理软件开发工具包](https://developers.google.com/admin-sdk/)(忽略传统的 G 套件品牌；这也适用于云身份 API)不是通过云控制台呈现的。

[Apache JMeter(TM)](http://jmeter.apache.org/) 是一个流行的开源负载测试工具；它包括用兼容 JSR223 的语言编写的可脚本化的采样器，比如 Groovy 和 BeanShell。

然而，在编写脚本时，您可能更喜欢用自己熟悉的语言编写代码。如果你对 [Go 编程语言](https://golang.org/)的发布很感兴趣，但一直在等待合适的用例，这就是它。

# JMeter

熟悉 JMeter 的人可能会发现为 Google Cloud 配置 JMeter 是显而易见的；然而，如果你是 JMeter 或 Google Cloud 的新手，以下内容可能会帮你省去一些试错的麻烦。

## 创建 OAuth 客户端 ID

如果您的项目有关联的非默认配额，则此步骤是必需的；否则可以跳过。

您需要一个 Google Cloud 凭证来授权 JMeter 执行您的项目；确保在您想要测试的项目上创建它，以便应用适当的配额。创建一个新的凭据，而不是重复使用现有的凭据，以防它被破坏。

*   应用类型:web 应用
*   授权的 Javascript 起源:[https://developers.google.com](https://developers.google.com)
*   授权重定向:[https://developers.google.com/oauthplayground](https://developers.google.com/oauthplayground)

安全地记录客户端 ID 和密码。

## 获取访问令牌

确保您处于与测试项目的域相关联的浏览器会话中，并且具有足够的权限来授权 API 范围的身份(例如，Admin SDK APIs 的超级管理员角色)。

在 [OAuth 2.0 游乐场](https://developers.google.com/oauthplayground) …

如果您的项目有非默认配额，请单击设置(齿轮图标)并…

*   选择“使用您自己的 OAuth 凭据”
*   提供您在上面生成的客户端 ID 和密码

否则你可以使用操场生成的默认凭证。

在主屏幕中，按照提示进行操作:

*   第一步。选择并授权 API:提供你要测试的 API 的范围，例如[https://www.googleapis.com/auth/admin.directory.group](https://www.googleapis.com/auth/admin.directory.group.readonly)API 的 https://www . Google APIs . com/auth/admin . directory . group . readonly。
*   第二步。令牌的交换授权码

这可能会将您直接带到步骤 3，并折叠步骤 2；如有必要，展开步骤 2 以查看访问令牌。

在后续步骤中，您将返回到此屏幕以获取访问令牌，因此不要关闭此窗口。

## 安装 JMeter

安装 JMeter (Java SDK 是记录 HTTPS 的[先决条件](https://jmeter.apache.org/usermanual/get-started.html#requirements)，所以如果还没有安装的话先安装它)。

*   Mac: brew 安装 JMeter

## 运行 JMeter

从安装的 bin 目录中，运行 JMeter 的 UI。

```
$ bash jmeter
```

## 配置 JMeter 测试计划

要么从[这个模板](https://gist.github.com/ferrisargyle/95e539a75fe720beca3688a69c1549ca)开始，只提供下面的粗体参数，要么创建一个新计划并…

添加一个[线程组](https://jmeter.apache.org/usermanual/test_plan.html#thread_group)(右击测试计划)并配置它:

*   添加->线程(用户)->线程组
*   发生采样器错误后要采取的操作:停止线程
*   线程属性->线程数量:10；从 1 开始，逐渐增加直到你到达你想要的 QPS。
*   循环计数:永远
*   调度程序配置->持续时间:100 秒；选择与您正在测试的配额期相匹配的持续时间。

添加一个 [HTTP 请求采样器](https://jmeter.apache.org/usermanual/component_reference.html#HTTP_Request)(右击线程组)并配置它:

*   添加->采样器-> HTTP 请求
*   Web 服务器->协议:https
*   Web 服务器->名称或 IP:content.googleapis.com
*   方法:GET(除非您正在测试插入，这通常需要脚本)
*   **路径**:追加到服务器，例如/admin/directory/v1/groups
*   **参数**(不编码):这些被附加到路径中，例如，domain=yourdomain.com，userKey=testuser@yourdomain.com

添加一个 [HTTP 头管理器](https://jmeter.apache.org/usermanual/component_reference.html#HTTP_Header_Manager)(右击测试计划)并配置它:

*   添加->配置元素-> HTTP 头管理器
*   **Headers**:Authorization = Bearer【您的访问令牌:复制/粘贴您之前在 OAuth 2.0 Playground 中创建的令牌；您可能需要先刷新它]

添加[总结报告](https://jmeter.apache.org/usermanual/component_reference.html#Summary_Report)和[结果树](https://jmeter.apache.org/usermanual/component_reference.html#View_Results_Tree) [监听器](https://jmeter.apache.org/usermanual/component_reference.html#listeners)(右击测试计划):

*   添加->监听程序->摘要报告
*   添加->监听器->查看结果树；确保未选中记录/仅显示框来查看错误和成功日志。

## 保存 JMeter 测试计划

在任何情况下都会提示您…

## 运行 JMeter 测试计划

选择摘要报告以监控进度，然后单击屏幕顶部的绿色播放按钮。

运行完成后，检查监听器:

*   摘要报告显示在配额达到最大值或运行超过配置的持续时间之前提交的查询总数(“样本数”)，以及 QPS(“吞吐量”)。
*   查看结果树->响应数据显示详细的响应和任何错误消息。

当您对您的测试计划感到满意时，您可以直接从命令行运行它。

# 去

负载测试和多线程就像烤面包和果酱一样；然而，Go 的性能不如 JMeter，所以你需要运行更多的线程，也就是 Go 例程，来生成与 JMeter 相同的负载；作为回报，你得到了大量的控制和灵活性。

有几种方法可以实现多线程:

*   [互斥](https://gobyexample.com/mutexes)
*   [有状态 goroutines 和通道](https://gobyexample.com/stateful-goroutines)

互斥体非常适合负载测试用例，因此您可以将示例互斥体应用程序与 [Admin SDK Go quickstart 应用程序](https://developers.google.com/admin-sdk/directory/v1/quickstart/go)混合，并添加少量 [CLI sugar](https://github.com/jawher/mow.cli) 来构建一个简单的负载测试工具，用于组列表和插入操作。

list 命令的操作类似于上面的 JMeter 测试；insert 命令在插入组名时会增加组名。

你需要用 *go build* 编译代码，然后运行二进制；命令行标志和参数对 *go run* 快捷方式不起作用。

## 下载 OAuth 凭证

您需要根据 Admin SDK 授权该工具(这需要使用与具有超级管理员角色的用户相关联的会话；Google 云平台 API 只需要通常的 IAM 权限)。

使用“[启用目录 API](https://developers.google.com/admin-sdk/directory/v1/quickstart/go) 按钮启用现有或新项目上的 API，并将令牌下载到您的源目录；该应用程序将带您通过 auth 流，而不是必须使用 OAuth 2.0 playground。