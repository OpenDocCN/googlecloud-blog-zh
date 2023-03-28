# 应用程序脚本中的 Google 云存储签名 URL

> 原文：<https://medium.com/google-cloud/google-cloud-storage-signed-urls-in-apps-script-161b74a9c5e7?source=collection_archive---------2----------------------->

## 简单、安全的图标

Apps Script 的两个关键挑战是存储非 G 套件数据，以及扩展 Apps Script 以利用更广泛的 Google 生态系统；先进的谷歌服务有助于解决后一个挑战。

其中一个最重要的地方是在你的 [HTMLService UI](https://developers.google.com/apps-script/guides/html/) 中显示图像:一旦你超越了简单的 UI，你会想要以图标的形式为你的用户提供图形提示。

[谷歌云存储](https://cloud.google.com/storage/) (GCS)是存储这些的天然场所，你可以简单地从你的 HTMLService 引用对象的 URL 只要对象是公共的。然而，让你从图标供应商(或你自己开发的)那里购买的图标库免费可用可能比你想提供的更像是一种公共服务，更不用说潜在的许可问题了。

您可以将 GCS 对象设为私有，只授予您域中的用户访问权限……只要您没有提供公开可用的 [G Suite 附加组件](https://developers.google.com/apps-script/add-ons/)，在这种情况下，您不知道您的用户是谁。这就是 [GCS 签名 URL](https://cloud.google.com/storage/docs/access-control/signed-urls)的用武之地。

GCS 签名的 URL 允许用户在有限的时间内读取、写入或删除该资源。但是，GCS 没有应用程序脚本高级服务；GCS 文档提供了一个[详细的方法](https://cloud.google.com/storage/docs/access-control/create-signed-urls-program)来滚动你自己的签名 URL，但是像所有的编码一样，有很多机会误入歧途。这就是下面这个函数的用处。

# 使用

在您的附加项目的 [GCloud 控制台](https://cloud.google.com/console/iam-admin/serviceaccounts)中创建一个服务帐户凭证并下载它。

将[字符串化的](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/JSON/stringify)凭证存储在您的附加组件的[脚本属性](https://developers.google.com/apps-script/guides/properties)中。

*   您没有将凭据存储在附加代码中，是吗？提示:在附加组件中编写一个脚本属性集包装函数，并从外部工具调用这个函数作为库方法来存储凭证…这允许您使用外部工具，而不用担心凭证在您的附加组件代码中持续存在，因为它经历了所有版本。

以安全的方式从您的计算机中删除下载的凭据，例如

```
$ shred -u ~/Downloads/<KEY_FILE>
```

向服务帐户授予对您的对象或存储桶的 reader 权限。

在您的 HTMLService 模板中，用 scriptlet 替换硬编码的图像引用，例如。

```
<img src='<?= GCSImg ?>' alt="Signed URL Image"/>
```

在创建 HTMLService 的函数中

*   从附加组件的脚本属性中获取凭证，[将其转换回 JSON 对象](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/JSON/parse)
*   使用适当的参数调用 getSignedURL_ function，并为签名的 URL 设置一个模板变量，例如。

```
template.GCSImg = getSignedURL_(GCSSignedURLCredential, GCSObjectURL,...);
```

# 下一步是什么

阅读以下内容，了解本文中描述的解决方案组件的更多信息:

*   [Apps 脚本高级谷歌服务](https://developers.google.com/apps-script/reference/)。
*   [Apps 脚本 HTMLService UI](https://developers.google.com/apps-script/guides/html/) 。
*   [谷歌云存储(GCS)签名网址](https://cloud.google.com/storage/docs/access-control/signed-urls)。

阅读以下指南，了解谷歌云平台在以下领域的能力。

*   [GCS 使用客户提供的加密密钥签署的 URL](/google-cloud/gcs-signed-url-with-customer-supplied-encryption-key-c89740f31855)。