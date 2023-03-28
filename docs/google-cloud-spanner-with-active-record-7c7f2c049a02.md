# 带活动记录的谷歌云扳手

> 原文：<https://medium.com/google-cloud/google-cloud-spanner-with-active-record-7c7f2c049a02?source=collection_archive---------2----------------------->

[Google Cloud Spanner](https://cloud.google.com/spanner) 是一个全面管理、可扩展的关系数据库服务，用于区域和全球应用数据。它是第一个可扩展的、企业级的、全球分布的、高度一致的数据库服务，专为云构建，将关系数据库结构的优势与非关系水平扩展相结合。

![](img/d8eb38f630f7dcea92344bcff4514caf.png)

[活动记录](https://guides.rubyonrails.org/active_record_basics.html)是 Ruby 的 ORM 实现。活动记录现在可以作为 Google Cloud Spanner 的对象关系映射器(O/RM ),使用它的[开源提供者](https://github.com/googleapis/ruby-spanner-activerecord)。本文将通过创建一个简单的 Ruby 应用程序来帮助您开始使用活动记录。

完整的示例应用程序可以在这里找到:[https://github.com/olavloite/spanner-activerecord-example](https://github.com/olavloite/spanner-activerecord-example)

# 配置扳手适配器和数据库

首先，通过将以下内容添加到您的 gem 文件 : `gem 'activerecord-spanner-adapter'`中，将 Spanner 活动记录适配器添加到您的项目中

然后，我们需要配置一个数据库，以便与 Spanner 活动记录适配器一起使用。我们在一个`database.yml`文件中这样做:

扳手活动记录的数据库配置示例

示例配置文件将连接到本地主机上的 Spanner 模拟器。删除配置文件中的`emulator_host`条目，以连接到一个真实的扳手实例。

# 创建数据库和表

通过在`db`文件夹中为您的项目创建一个初始迁移，可以很容易地在您的数据库中创建表。

01_create_tables.rb

示例迁移文件将创建两个表；`singers`和`albums`。请注意:

1.  表格在`ddl_batch`块中定义。DDL 批处理大大减少了多个 DDL 语句的执行时间，强烈建议对包含多个 DDL 语句的所有迁移使用 DDL 批处理。
2.  `albums`表使用外键约束引用`singers`表。
3.  `singers`表为歌手的全名定义了一个生成的列。该列的值总是经过计算的，不可能将其他值写入该列。
4.  两个表都定义了一个`last_updated`列。这些列可以用最后更新记录的事务的提交时间戳来更新。

# 用测试数据填充数据库

您可以为数据库的初始(测试)数据使用一个`seeds.rb`文件。

然后，您可以使用`rake db:seed`命令播种您的数据库。在示例应用程序中，这是由 Rakefile 自动完成的。

注意种子文件中的`last_updated: :commit_timestamp`哈希值。将标记有`allow_commit_timestamp`的字段设置为符号`:commit_timestamp`将指示 Spanner 活动记录适配器将数据库中记录的值设置为插入或更新记录的事务的提交时间戳。

# 运行应用程序

您可以通过在项目的根目录下执行以下命令来运行示例应用程序:

`bundle exec rake run`

这将调用以下脚本:

用于运行扳手活动记录示例应用程序的 Rakefile

该脚本中的步骤是:

1.  下载并启动 Docker 容器中的[扳手模拟器](https://cloud.google.com/spanner/docs/emulator)
2.  在模拟器上创建一个测试实例和测试数据库。
3.  执行我们为示例应用程序定义的迁移。
4.  对数据库执行 db:seed 命令。这将在 seeds.rb 文件中插入测试数据。
5.  运行示例应用程序。

示例应用程序将执行几个查询，并使用 ActiveRecord 更新一些数据。完整文件请看一下[https://github . com/olavloite/spanner-active record-example/blob/main/application . Rb](https://github.com/olavloite/spanner-activerecord-example/blob/main/application.rb)文件。