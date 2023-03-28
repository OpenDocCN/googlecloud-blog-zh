# 一种检查授予您的云平台云存储桶的 IAM 权限的轻量级方法

> 原文：<https://medium.com/google-cloud/a-lightweight-way-to-check-the-iam-permissions-granted-to-your-cloud-platform-cloud-storage-buckets-803dd86ce9ca?source=collection_archive---------0----------------------->

了解您授予您的云存储桶的权限非常重要，这样您可以避免无意中授予太广泛的权限，因此请通过阅读[此](https://cloud.google.com/storage/docs/access-control/iam#roles)来了解 IAM 角色，因为它们与云存储有关。

这里[列出了云存储 IAM 的最佳实践](https://cloud.google.com/storage/docs/access-control/iam#best_practices)，但是为了验证您在设置权限时没有犯错误，轻量级检查可能是有用的。

这就是这篇文章的内容。对于框架方法，你可能想要检查可扩展的[凡赛堤安全性](http://forsetisecurity.org/about/)(我将在某个时候再次深入讨论)

你可以利用 GCP 提供的一个被低估的功能来快速检查一个桶的权限，这个功能允许你在浏览器中测试 API。

为此，您需要使用两个 API 页面:

如果你通过了认证，并且至少拥有项目的角色/查看者权限，这将会给你一个项目中所有存储桶的列表

现在您已经有了一个 bucket 列表，您可以通过使用[bucket:getiampolicy](https://cloud.google.com/storage/docs/json_api/v1/buckets/getIamPolicy)API 文档来获得针对所标识的 bucket 的权限。对于此页面，您至少需要对包含存储桶的项目拥有[storage . buckets . getiampolicy](https://cloud.google.com/storage/docs/access-control/iam-permissions)权限

bucket 上的权限作为 JSON 返回(耶！)对于我的一个桶(适当的匿名输出)来说，它看起来像这样:

```
{“kind”: “storage#policy”,“resourceId”: “projects/_/buckets/mydemobucket”,“bindings”: [{“role”: “roles/storage.legacyBucketOwner”,“members”: [“projectEditor:my-project-id”,“projectOwner:my-project-id”]},{“role”: “roles/storage.legacyBucketReader”,“members”: [“projectViewer:my-project-id”]},{“role”: “roles/storage.objectViewer”,“members”: [“user:joe@joesworkdomain”]}],“etag”: “CAM=”}
```

不过这有点乏味(因为每个 bucket 都有单独的 http 请求),而且我喜欢自动化工作并使用易于使用的[客户端库](https://cloud.google.com/apis/docs/cloud-client-libraries),我写了一个小 python 脚本，您可以使用它作为基础来自动检查项目中 bucket 的权限。它也给你一个你可以访问的项目列表(只是因为真的)。

它将每个 bucket 的权限写入一个文本文件，但是可以很容易地扩展到将值写入 CloudSQL 表或 BigQuery。(我倾向于将 list project 函数分解到一个单独的脚本中，并使用输出的 projectID 来设置 GOOGLE_CLOUD_PROJECT env 变量，然后调用一个脚本为该项目进行存储桶检查)

该脚本针对一个存储桶为每个权限写一行，格式如下:

```
<Bucket: BUCKETNAME> : Role: roles/storage.BUCKET-IAM-ROLE, Members: set([u’GROUP or USER:my-project-id’,u’GROUP or USER:my-project-id’,])
```

如果相同的角色被授予其他组或用户(GCP IAM 术语中的成员),则该条目将被包括在内，如上所示分开。

我的一个桶的示例(适当的匿名输出)如下所示:

```
<Bucket: mydemobucket> : Role: roles/storage.legacyBucketOwner, Members: set([u’projectEditor:my-project-id’, u’projectOwner:my-project-id’])<Bucket: mydemobucket> : Role: roles/storage.objectViewer, Members: set([u’user:joe@joesworkdomain’])<Bucket:mydemobucket> : Role: roles/storage.legacyBucketReader, Members: set([u’projectViewer:my-project-id’])
```

请注意，输出是敏感的，因此请确保您采取适当的步骤来注意如何管理数据以及您授予谁访问数据的权限。

你可以在这里找到我的基本 python 脚本(是的，我知道没有测试，但这是一个非常简单的脚本&我想在我休假一周后回来工作之前扭转这个局面)。

该脚本的灵感来自 Ric Harvey( @ric_harvey)，他在 github 上发布了一个方便的 s3 权限检查器。谢谢里克。

我的没有 Ric 的输出漂亮，虽然它真的是为你设计的，不仅仅是写文本文件