# 鲁比& GCP:上传图片到云存储

> 原文：<https://medium.com/google-cloud/ruby-gcp-uploading-pictures-to-cloud-storage-4c136dd2405f?source=collection_archive---------0----------------------->

向存储提供商上传内容是我在云提供商那里做的最基本的任务之一。至少应该是。这是一个用 Ruby 上传文件到 Google 云存储的快速教程。本教程的代码是[这里是](https://github.com/thagomizer/examples/tree/master/CloudMinute/Ruby)。

# 设置

1.  要运行本教程，你需要一个谷歌云平台账户和一个项目。您可以在 http://cloud.google.com/free-trial 的[创建账户和项目。](http://cloud.google.com/free-trial)
2.  你还需要谷歌云存储宝石。gem 安装谷歌-云存储
3.  在 [API 管理器](https://console.cloud.google.com/apis)中启用存储 API。你希望启用 Google 云存储和 Google 云存储 JSON API。
4.  最后，您需要一个服务帐户。

*   从 console.cloud.google.com 点击左侧菜单中的“API 管理器”。
*   加载 API 管理器页面后，单击“凭据”。它可能只是作为一个关键图标出现。
*   单击“创建凭据”并选择“服务帐户密钥”。
*   您可以使用现有的服务帐户或创建一个新帐户。它应该具有存储对象管理员角色。您可能需要向下滚动角色下拉列表，以查看存储类别。
*   密钥应该会自动下载。把它放在安全的地方。我将它放在我的项目的根目录中，并将其重命名为 service-account.json。

# 代码

上传文件的代码很简单。

首先，需要库，并使用您的项目 id 和服务帐户密钥创建一个新的 gcloud 对象。您可以从云控制台获取项目 id。它可能与您的项目名称不同。

然后使用 gcloud 对象创建一个存储对象。

之后，创建一个云存储桶。云存储空间名称在整个 Google Cloud 中必须是唯一的。存储桶名称也可能是全局可见的，因此它们不应该包含专有信息。更多信息参见[存储桶命名要求](https://cloud.google.com/storage/docs/naming#requirements)和[存储桶命名最佳实践](https://cloud.google.com/storage/docs/best-practices)。

在这个例子中，这个桶被称为 my_goat_pictures。

一旦创建了 bucket，就可以使用 create_file 方法上传文件。在示例中，我上传了一张名为 goat.jpg 的图片，它在云存储桶中保存为 uploaded_goat.jpg。

您可以使用 web 控制台中的存储浏览器来验证文件是否已正确上传。[https://console.cloud.google.com/storage/browser](https://console.cloud.google.com/storage/browse)

# 高级选项

创建存储桶时，您可以指定访问控制列表(ACL)、存储桶的位置以及要使用的存储类别。您还可以为整个存储桶打开对象版本控制。如果您使用 bucket 来服务一个静态网站(可能是用 Jekyll 生成的)，那么您可以在创建 bucket 时为站点和 404 页面设置索引/主页。

同样，在创建文件时可以指定许多选项。可以指定缓存控制信息、内容元数据(编码、语言、处置)、自定义元数据和加密密钥。您还可以包括校验和，只有当校验和与云存储计算的值匹配时，才会创建文件。

有关云存储 API 的更多信息，请查看[文档](http://googlecloudplatform.github.io/google-cloud-ruby/#/docs/google-cloud-storage/master/google/cloud/storage)。

09/29/16

*原载于 2016 年 9 月 30 日*[*www.thagomizer.com*](http://www.thagomizer.com/blog/2016/09/30/ruby-gcp-uploading-pictures-to-cloud-storage.html)*。*