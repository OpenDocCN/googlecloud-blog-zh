# 什么是 App Engine Gem

> 原文：<https://medium.com/google-cloud/what-is-the-app-engine-gem-75fe64ed9319?source=collection_archive---------0----------------------->

我真的很喜欢和谷歌的 Ruby 团队一起工作。我们的目标之一是为社区提供易于使用的工具和产品。谷歌对 Ruby 社区以及我们所有令人愉快的怪癖的了解并不多。所以作为一个团队，我们花了一部分时间为我们的 Ruby 社区做宣传，并向谷歌员工传授 Ruby 的乐趣。我们还花一些时间想出创造性的方法来满足我们社区的独特需求。

这些创造性的解决方案之一是[应用引擎 gem](https://rubygems.org/gems/appengine) 。App Engine gem 提供了一组工具，使得在 Ruby 上使用 App Engine 变得更加容易，尤其是在 Rails 上。在 Rails 上使用 App Engine 并不困难，我们知道人们已经在这么做了。事实上，我已经在我的一些项目中使用了它，但是我们知道我们可以做得更好。

App Engine gem 提供的第一件事是 Rails 应用程序的自动 Stackdriver 集成。对于大多数语言社区来说，必须自己进行配置和认证来获得日志和崩溃报告之类的东西并不是什么大事。Rails 通过 [railties](http://api.rubyonrails.org/v5.1/classes/Rails/Railtie.html) 支持第三方库的“不需要代码”集成。这意味着当您使用 App Engine gem 时，除了将 gem 添加到您的 gem 文件中之外，您不必做任何其他事情，其余的初始化工作将由您来完成。

对于 Ruby 爱好者来说，这看起来很正常，但是当我在 Ruby 和 Stackdriver 上写我的[博客时，一个评论者问:“安装代码在哪里？”我告诉他们你不需要，他们很惊讶。有时候我会忘记 Ruby 和 Rails 对于像我这样懒惰的开发人员来说有多棒。](https://cloudplatform.googleblog.com/2017/10/now-you-can-monitor-debug-and-log-your-Ruby-apps-with-Stackdriver.html)

App Engine gem 带给您的另一个主要东西是一种针对生产环境运行应用程序命令的方式。当我们在春天发布 Ruby 的 App Engine 时，我们得到的最常见的问题之一是“但是我如何运行迁移？”此时，推荐的方法是将您的本地环境连接到生产数据库，并在本地运行 rake 命令。这是我在 2007 年第一次使用 Rails 时经常做的事情。我知道那时候这是个坏主意，但是很多人都这么做了。随着我们作为一个社区的成熟，我们已经正确地远离了这样的解决方案，以至于许多当前的 Rails 开发人员甚至不知道您可以这样做。

App Engine gem 为您提供了`rake appengine:exec -- [your command here]`，以便您可以在生产实例上运行迁移和其他 rake 任务。我们的 [Ruby 登录页面](http://cloud.google.com/ruby)上的例子已经更新为使用这个工具，所以你不再需要将你的本地开发环境连接到生产环境。是的，我们知道最初的计划不是一个好主意，这就是为什么我们发布了这个 gem 来让事情按照 Ruby 社区的方式运行。

如果你使用 App Engine 是为了工作还是为了副业，我鼓励你去看看 App Engine gem。

10/17/17

*原载于 2017 年 10 月 17 日【www.thagomizer.com】[](http://www.thagomizer.com/blog/2017/10/17/what-is-the-app-engine-gem.html)**。***