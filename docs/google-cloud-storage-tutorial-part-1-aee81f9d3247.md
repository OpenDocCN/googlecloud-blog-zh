# Google 云存储教程第 1 部分

> 原文：<https://medium.com/google-cloud/google-cloud-storage-tutorial-part-1-aee81f9d3247?source=collection_archive---------0----------------------->

![](img/4db4ae9df03a8d61537b7803ab4363ec.png)

在过去的几天里，我一直在钻研谷歌云存储(GCS)。你可能会认为这是谷歌云平台的一个无聊的方面，因为云存储已经存在了一段时间，但我在探索中发现了一些惊喜。让我们从一个简单的概述开始:

*   [**谷歌云存储**](http://cloud.google.com/storage) 不是 [**谷歌驱动**](https://www.google.com/drive/) 。Google Drive 是一种面向消费者的文件存储和同步服务，允许用户存储文档、共享文件以及与合作者编辑文档，并且它具有允许开发人员扩展它的 API。谷歌云存储是面向开发者和系统管理员的文件存储解决方案，具有允许应用程序存储和检索对象的 API。它具有全局边缘缓存、版本控制、OAuth 和粒度访问控制、可配置的安全性等特性..
*   谷歌云存储建立在谷歌庞大的全球基础设施之上。想想谷歌提供的所有文件，包括 YouTube 视频等。规模之大，尺度之大，超出了大多数人的理解。我最近参观了许多谷歌数据中心中的一个(这是非常难得的待遇，即使对谷歌人来说也是如此！)，这让我对谷歌的巨大规模有了一点了解，它彻底改变了我对“可扩展”和“快速”的定义。
*   谷歌云存储有一个基本功能的 web 界面，但我更喜欢使用命令行工具 [gsutil](https://cloud.google.com/storage/docs/gsutil) 。此外，命令行工具公开了更多的功能。还有[API(XML 和 JSON)](https://cloud.google.com/storage/docs/concepts-techniques) ，我们将在后面的文章中介绍。

## 初始设置:

*   如果你还没有谷歌云平台账户，你可以在[http://cloud.google.com](http://cloud.google.com/)创建一个。如果你以前从未注册过，你可以获得 60 天 300 美元的免费试用(截至本文撰写之时)。
*   安装 http://cloud.google.com/sdk[的谷歌云平台 SDK。](http://cloud.google.com/sdk)
*   [创建一个项目，并为该项目激活 Google 云存储](https://cloud.google.com/storage/docs/signup)(项目是 Google 云资源的逻辑分组——存储、服务、用户、实例等。).创建一个项目需要一个 web 浏览器，但是一旦你创建了它，你就可以使用命令行工具做几乎所有的事情。记下您的项目 ID。在本文的剩余部分，我将使用“ **gwprojectid** ”。
*   我们需要将我们的新项目设置为 Google Cloud 命令行工具的默认项目，否则我们将不得不对每个命令使用“ **-p gwprojectid** ”。要设置默认项目，请键入:**g cloud config set project GW projectid**

## 使用 gsutil 工具

下面是一个简单的存储和检索文件的例子。首先，我们需要创建一个 bucket——用于存储文件的基本容器。因为所有的存储桶名称共享全局名称空间，所以它需要在整个 Google Cloud 中是唯一的。当我们开始通过 http 公开共享文件时，你会在后面的文章中看到为什么会这样。对于这个例子，我将使用一个名为“gwbucket”的桶。引用存储桶时，您需要指定 URL，例如**GS://【bucket name】**。

```
#Create a bucket:
**gsutil mb gs://gwbucket**#Copy some files to the bucket:
**gsutil cp *.jpg gs://gwbucket**#List the files in the bucket:
**gsutil ls -l gs://gwbucket**#Copy a file from our bucket back to the local /tmp directory
**gsutil cp gs://gwbucket/sunset.jpg /tmp**#Delete the files:
**gsutil rm gs://gwbucket/***#Remove the bucket:
**gsutil rb gs://gwbucket**
```

还没有什么太令人兴奋的，但是您很快就会发现这些命令很熟悉。您可以将“-r”添加到大多数命令中，使其递归，如预期的那样。在[https://cloud.google.com/storage/docs/gsutil](https://cloud.google.com/storage/docs/gsutil)或从命令行使用 **gsutil help cp** 可获得所有命令的详细帮助

下面是另一个使用 [rsync](https://cloud.google.com/storage/docs/gsutil/commands/rsync) 和[版本控制](https://medium.com/p/aee81f9d3247/edit)的例子:

```
#Create a new bucket:
**gsutil mb gs://gwnewbucket**#Turn on versioning for the bucket
**gsutil versioning set on gs://gwnewbucket**#rsync the current directory to our new bucket
#Adding -m to run multiple parallel processes (speed boost)
**gsutil -m rsync -r -d . gs://gwnewbucket**#List all of the files in the bucket:
**gsutil ls -lr gs://gwnewbucket**
```

[了解更多关于对象版本控制的信息](https://cloud.google.com/storage/docs/object-versioning)
[了解更多关于-m 选项的信息](https://cloud.google.com/storage/docs/gsutil/addlhelp/TopLevelCommandLineOptions)

注:以上示例使用默认时段分类“标准”和默认时段位置“美国”。要了解其他选项，请参见[这些文档](https://cloud.google.com/storage/docs/gsutil/commands/mb#bucket-storage-classes)。

我使用了上面的最后一个例子(版本控制行除外)将我的 120，000+图像库备份到 Google 云存储中。每当我添加/修改/删除图像时，我简单地重复 **gsutil rsync。**

*(在 rsync 上使用-d 选项时要非常小心，因为它会从目标中删除任何已经从源中删除的文件。如果你有任何疑问，我建议使用< strong > -n < /strong >选项。-n 选项使 rsync 以“模拟运行”模式运行，即只输出将要复制或删除的内容，而不进行任何复制/删除。)*

在我的下一篇文章中，我将展示如何设置 bucket 对象生命周期管理，以便在对象达到一定年龄时配置自动删除，或者简单地保留特定对象的最后 n 个版本。当进行定期备份时，例如每晚归档日志文件等，这成为一个关键特性。然后，我将向您展示如何通过 HTTP 公开共享对象，以及如何利用 Google 的全球边缘缓存来提供非常快速的文件下载。