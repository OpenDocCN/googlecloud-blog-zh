# Firebase:在 Admin SDK 中将配置与代码分离

> 原文：<https://medium.com/google-cloud/firebase-separating-configuration-from-code-in-admin-sdk-d2bcd2e87de6?source=collection_archive---------1----------------------->

![](img/3792090230a81d453c78befc8e2a0535.png)

[Firebase Admin SDK](https://firebase.google.com/docs/admin/setup)用于从可信的服务器端环境访问 [Firebase](https://firebase.google.com/) 服务。然而，在访问任何 Firebase 服务之前，必须用一组有效的 Google 凭证初始化 Admin SDK。谷歌证书有几种形式，它们授权应用程序访问 Firebase 和其他各种[谷歌云平台(GCP)](https://cloud.google.com) 服务。

[服务账户凭证](https://cloud.google.com/compute/docs/access/service-accounts)是最常用的谷歌凭证类型之一。这些凭证可以作为 JSON 文件从 GCP 控制台或 Firebase 控制台下载，并显式地传递给 Admin SDK，如清单 1 所示。

清单 1:用服务帐户初始化 Admin SDK

虽然这种方法开始起来相当简单，但从长远来看，它会带来一些挑战:

1.  服务帐户凭证必须与代码一起打包和部署，这使得相关的 DevOps 过程变得复杂。
2.  服务帐户凭据文件的意外泄露可能会给应用程序和数据安全带来严重后果。

因此，使用隐式形式的凭证通常是个好主意，比如 Google [应用程序默认凭证(ADC)](https://developers.google.com/identity/protocols/application-default-credentials) 。ADC 使开发人员不必将有形的凭证文件与代码一起发送。相反，ADC 使应用能够从其运行时环境中自动*发现*凭证。由于在部署过程中没有可能意外暴露的敏感资产，ADC 还可以提高部署的应用程序的安全级别。

## 使用应用程序默认凭据

当应用程序代码部署在 GCP 环境(如应用程序引擎、计算引擎和云功能)中时，ADC 发现即开即用。对于本地测试，有两种方法可以用 Google 凭证播种环境，以便 ADC 发现能够成功:

1.  将服务帐户凭证文件保存到本地文件系统中的安全位置，并设置指向该文件的`GOOGLE_APPLICATION_CREDENTIALS`环境变量。
2.  安装 [Google Cloud SDK](https://cloud.google.com/sdk/) ，执行`gcloud auth application-default login`命令。

第一种方法既适合本地测试，也适合 GCP 以外的生产部署。后者严格用于本地测试。一旦以这两种方式之一设置了开发环境，就可以用 ADC 初始化 Firebase Admin SDK，如清单 2 所示。

清单 2:用应用程序默认凭证初始化 Admin SDK

请注意，我们不再需要将文件路径硬编码到应用程序中。一旦您在本地测试了您的代码，您可以简单地将它转移到 GCP，在那里它将继续工作，不需要任何额外的配置。

使用 ADC 时，有一点值得注意。当用 ADC 初始化 Firebase Admin SDK 时，涉及加密签名数据的功能不起作用。[定制令牌铸造](https://firebase.google.com/docs/auth/admin/create-custom-tokens)和[为云存储对象生成签名的 URL](https://cloud.google.com/storage/docs/access-control/create-signed-urls-program)是这种特性最显著的特征。签名数据需要私钥，目前 Admin SDK 只能从服务帐户凭据获取私钥。这一限制可能会在未来得到解决，从而使 ADC 的使用更加不受限制。

## 从环境中加载其他 Firebase 选项

ADC 使我们能够从应用程序中取出 Google 凭证。但是其他的 Firebase 设置呢？清单 2 显示我们仍然需要将 Firebase 数据库 URL 硬编码到应用程序中。其他 Firebase 选项也是如此，比如云存储桶和 GCP 项目 ID。如果 Admin SDK 可以从环境中自动发现这些参数，这样您就可以将代码从配置中完全分离出来，这不是很好吗？事实上，现在你可以！

Firebase 团队刚刚[发布了](https://firebase.google.com/support/releases#january_11_2018)Admin SDK 中的一个新特性，允许从环境中加载所有 Firebase 配置选项。只需创建一个类似清单 3 的 JSON 文件，包含必要的设置，并设置环境变量`FIREBASE_CONFIG`指向它。

清单 Admin SDK 的示例 Firebase 配置文件

现在可以不用任何参数初始化 Firebase Admin SDK，如清单 4 所示。

清单 4:在没有显式参数的情况下初始化管理 SDK

这指示 SDK 从环境中获取 Firebase 选项，同时使用 ADC 进行授权。类似的 API 现在可以在 Admin SDK 支持的所有编程语言中使用(Node.js、Java、Python 和 Go)。然而，这是一个新特性，GCP 还不支持开箱即用。但是您可以在您的开发环境中使用它，并在您认为合适的时候在非 GCP 部署 Firebase Admin SDK。

如果您希望在 GCP 立即使用这个特性，您可以通过在您的云实例中设置`FIREBASE_CONFIG`环境变量来实现。这在计算引擎和[应用引擎](https://cloud.google.com/appengine/docs/flexible/nodejs/configuring-your-app-with-app-yaml#Node.js_app_yaml_Defining_environment_variables)中很容易实现。如果没有该变量，您的应用程序将不会自动发现 Firebase 选项，但 ADC 发现将照常工作。因此，目前这一功能可能看起来不如 ADC 强大。然而，它提供了一个简单的和语言中立的机制来将 Firebase 选项从应用程序代码中分离出来。随着时间的推移，随着它在 GCP 变得标准化，它将会得到自己的认可。

## 结论

将配置从代码中分离出来是一种广泛使用的软件工程实践，它可以使代码更简单、更易于维护。有了应用程序默认凭证和新的 Admin SDK 初始化 API，开发人员可以清楚地将 Google 凭证和 Firebase 选项从他们的应用程序中分离出来。我希望这篇文章鼓励更多的开发人员利用这些特性，这简化了开发，同时使应用程序在不同的环境中更加可移植。愉快的编码，让我知道你对新的 Admin SDK 初始化 API 的看法。