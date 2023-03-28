# 自动化和安排 Linux 文件系统与 Google 云存储的同步

> 原文：<https://medium.com/google-cloud/automating-and-scheduling-a-linux-filesystem-sync-with-google-cloud-storage-576a07ca2fdb?source=collection_archive---------1----------------------->

**TL；DR:** 我学会了如何使用[存储转移服务](https://cloud.google.com/storage/transfer/)将我的遗留文件系统数据迁移到[谷歌云存储](https://cloud.google.com/storage)。在下面[找到我写的一个脚本](https://gist.github.com/allenday/88a63ed80f9b4fda4080704ac034698c)，它将帮助你做同样的事情。

**动机:**我有一堆分散在各种服务器和托管服务上的内容，因为几十年来我一直在网上放东西——远在云普遍可用之前。设置通常是 Linux 文件系统，主机已经有一个 HTTP 服务器，或者我可以设置一个。这些服务器的一些文件系统仍然在添加/删除文件，我希望有一种方法来获得这个过程的结果，即文件系统的内容，转移到 Google 云存储中。这为应用程序提供了一条向前迁移的路径，因为我最终可以迁移修改文件系统、使用 CDN 等的服务。

原来 Google Cloud 提供了一个存储传输服务，除了支持从 AWS S3 桶导入之外，还有一种从 HTTP 服务器加载数据的方法。您可以在这里阅读有关传输服务:[https://cloud.google.com/storage/transfer/](https://cloud.google.com/storage/transfer/)以及与本文档相关的，使用 TsvHttpData 格式描述通过 HTTP 导入的对象的协议:[https://cloud.google.com/storage/transfer/create-url-list](https://cloud.google.com/storage/transfer/create-url-list)

简言之，TsvHttpData 需要一个包含以下内容的 3 列 TSV 文件:

1.  对象的 URL
2.  对象的大小，以字节为单位
3.  对象的 MD5 校验和

然后，您可以使用 gcloud 创建一个计划作业来检索 TSV 文件并导入任何丢失/更新的对象。

我编写了一个脚本，它可以递归遍历文件的子目录，找到自创建 TsvHttpData 文件的上一个版本以来修改过的文件，并用新信息更新该 TsvHttpData 文件(新创建的文件、不再存在的省略文件、修改文件的更新 MD5 校验和)。它做了正确的事情来减少 I/O，不重新计算没有改变的文件的 MD5 校验和。以下是 GitHub 要点中的源代码:

接下来要做的是将这个脚本放在 cron 上，以及删除最后修改的时间戳的后处理步骤，我将这个时间戳作为额外的一列添加到 TsvHttpData 格式中(以节省 I/O)。这两个命令看起来像:

```
#create the URL list, sizes, checksums, and last-modified timestamps
make_TsvHttpData.pl http://my.org/ ~/public_html/sync/ ~/THD.pre#write the TsvHttpData file to a location visible over HTTP:
cat ~/THD.pre | cut -f 1,2,3 > ~/public_html/THD.tsv
```

请让我知道❤或评论，如果你觉得这很有用！