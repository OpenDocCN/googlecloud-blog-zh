# Firebase:服务器端的故事

> 原文：<https://medium.com/google-cloud/firebase-the-server-side-story-1f89064e9c16?source=collection_archive---------0----------------------->

![](img/dbc43ee640f5b06c4937b1ac861b62ba.png)

正如大多数开发者所知，许多网络和移动应用并不存在于真空中。相反，它们是一个更大的软件生态系统的一部分，包括其他应用程序、后端服务、数据库、API 甚至遗留系统。因此，一个应用程序的成功往往取决于它与各种其他软件无缝集成的能力。对于一个 Firebase 应用程序开发者来说，这意味着能够用 Firebase 集成多个异构系统，并在需要时在它们之间共享数据。

[Firebase Admin SDK](https://firebase.google.com/docs/admin/setup)正是考虑到这一关键需求而设计的。简而言之，管理 SDK 有助于从*可信环境*访问 Firebase 服务。这里的术语可信环境表示 Firebase 应用程序的开发人员和管理员所信任的任何编程或运行时环境。因此，Admin SDKs 通常使用属于 Firebase 项目本身的凭证进行初始化。此外，管理 SDK 从不部署在最终用户设备上，从应用程序开发人员的角度来看，这是不可信的。

## 谁是利益相关者？

在我们进一步讨论之前，我们需要清楚地确定与 Firebase 应用程序相关的两类用户。

*   **App 开发者** — 是指实施、发布和维护一个 Firebase app 的人。这是一个由程序员、系统管理员和各种其他 DevOps 人员组成的小型但多样化的团队。
*   **最终用户** —指通过移动设备和/或网络浏览器下载、安装和使用 Firebase 应用程序的人。一个病毒式传播的应用程序可能拥有数百万的最终用户。

*应用开发者*使用 Firebase 客户端 SDK 构建应用。Firebase 为 iOS、Android 和 Web (JS)平台提供客户端 SDK。一旦这样的应用程序发布，*最终用户*通过他们的移动设备和网络浏览器消费它。必要时，他们使用自己的凭据(如用户名+密码、谷歌账户、脸书账户)登录应用程序。

另一方面，使用 Firebase Admin SDKs 构建的软件由*应用开发者*自己部署和运行。他们决定这些软件的用途，以及如何向最终用户公开这些软件。基于管理 SDK 的软件通常补充 Firebase 应用程序，提供一些后端功能或支持与其他系统的集成。

## 您可以在哪里部署管理 SDK？

Firebase Admin SDKs 可以部署在应用程序开发人员可以控制的任何环境中。典型的例子包括:

1.  由应用程序开发人员运行和管理的应用服务器，可能位于他们自己的数据中心。
2.  开发者正在使用的云平台环境(例如由开发者启动的[谷歌应用引擎](https://cloud.google.com/appengine/)或[计算引擎](https://cloud.google.com/compute/)实例)。
3.  开发者用来编程的无服务器框架(例如[谷歌云功能](https://cloud.google.com/functions/))。

请注意，在上述所有情况下，应用程序开发人员控制环境，以及软件在每个环境中的部署方式。例如，应用程序开发人员可以随时自由更改、重新部署或终止他们的软件。应用程序开发人员还决定在这样的环境中部署时如何保护他们的代码和相关的交互。因此，他们可以*信任*环境不会被最终用户破坏。这就是这些环境适合 Admin SDK 的原因。

相比之下，最终用户移动设备和 web 浏览器不在 Firebase 应用程序开发人员的控制之下，因此不应该考虑进行 Admin SDK 部署。在极端情况下，最终用户甚至可能修改在这些环境中运行的代码，因此应用程序开发人员无法依赖代码始终在可接受的参数内运行。然而，最终用户设备和应用与 Admin SDK 代码进行远程交互是完全正常的，也是意料之中的。Admin SDKs 提供了一种安全地[授权](https://firebase.google.com/docs/auth/admin/verify-id-tokens)此类远程客户端交互的方法。毕竟，集成不同的系统是管理 SDK 的目的。

*我打算在不久的将来与一位同事合作撰写一篇关于这个主题的更详细的文章。因此，我现在将避免进入一个详细的讨论，并恳请你保持关注。*

## 你能用管理 SDK 做什么？

管理 SDK 放宽了 Firebase 客户端 SDK 中存在的一些安全措施，同时提供了对实现开发人员和管理员用例有用的 API。一些流行的管理 SDK 用例有:

*   为 Firebase 应用程序开发管理仪表板。
*   为 Firebase 应用程序构建关键的后端服务。
*   设置用户帐户、管理用户角色和设置权限。
*   与第三方用户认证系统集成。
*   向应用程序的用户发送通知。
*   归档或汇总由应用程序管理的数据。
*   跨系统迁移数据。

诸如此类的用例可以以许多不同的形式实现，包括 web 应用、批处理作业、无服务器事件处理程序和命令行工具。因此，Firebase Admin SDKs 经常被用来开发各种各样的软件。此外，在撰写本文时，Firebase Admin SDKs 有四种编程语言版本——node . js、Java、Python 和 Go。这使得 Firebase 能够集成各种系统。

## 正在初始化 Firebase 管理 SDK

Firebase Admin SDKs 通常使用两种类型的凭证之一进行初始化。

*   [**服务账户凭证**](https://developers.google.com/identity/protocols/OAuth2ServiceAccount) —可用于授权服务器间交互的凭证。每个服务账户凭证属于一个特定的[谷歌云平台](https://cloud.google.com/) (GCP)项目，只有项目的开发者和管理员才能获得。因为 Firebase 项目只是伪装的 GCP 项目，所以服务帐户凭证也可以用于授权 Admin SDK 进行的 Firebase 交互。Firebase 项目的服务帐户凭证可以作为 JSON 文件从 [Firebase 控制台](https://console.firebase.google.com)或 GCP 控制台下载。

清单 1:用服务帐户凭证初始化(Java)

*   **Google** [**应用默认凭证**](https://developers.google.com/identity/protocols/application-default-credentials) **(ADC)** —一种凭证，可随时用于部署在托管环境中的代码，如 Google App Engine、Google Compute Engine 和 Google Cloud 函数。有了 ADC，应用程序开发人员不必在代码中显式打包任何凭据。相反，它们只是在代码中指示应用程序应该使用 ADC，应用程序将通过检查其运行时环境来*发现*适当的凭证。在幕后，ADC 通常只是 GCP 为运行时环境隐式提供的服务帐户凭证。

清单 2:用 Google 应用程序默认凭证初始化(Java)

Admin SDKs 还支持第三种类型的凭证，这允许使用 OAuth2 刷新令牌初始化 SDK。虽然前两种类型的凭证属于 Firebase 项目，但刷新令牌凭证通常属于个人开发人员。但是这在实践中很少使用，除了在本地测试应用程序。

如果您的代码部署在 Google 管理的环境中，强烈建议您使用应用程序默认凭证初始化 Admin SDK。对于所有其他环境(本地虚拟机、AWS、Azure 和其他)，使用服务帐户凭据。但是，在少数情况下，即使部署到 GCP，也必须使用服务帐户凭据。这通常是在使用 Admin SDK 对一些数据进行加密签名的时候。典型的例子包括:

*   [创建自定义 JWT 令牌](https://firebase.google.com/docs/auth/admin/create-custom-tokens)
*   [为谷歌云存储对象创建签名的 URL](https://cloud.google.com/storage/docs/access-control/create-signed-urls-program)

这些用例需要一个私钥来签署数据。目前，Admin SDK 只能从明确指定的服务帐户凭据中获取私钥。这在某种程度上是 Admin SDK 的一个限制，可能会在未来的版本中得到解决。

另一个相关的注意事项是，请记住，服务帐户凭证不应该与不是 Firebase 项目开发人员或管理员的人共享。服务帐户允许访问与 Firebase 项目相关的许多资源。如果落入坏人之手，它们会导致严重的问题，从轻微的资源滥用到全面的 DoS 攻击。小心处理这些 JSON 文件，永远不要将它们签入公共版本控制。缺少显式凭证文件(可能会意外泄露)是 ADC 相对于服务帐户的一个主要优势。

## 它的“管理”是什么？

Firebase Admin SDKs 提供对 Firebase 服务和相关项目资源的不受监管的管理访问。把它想象成在[超级用户](https://en.wikipedia.org/wiki/Superuser)模式下操作。因此，它可以在 Firebase 项目中执行某些特权操作，例如删除用户帐户或重置用户密码。另一方面，Firebase 客户端 SDK 在最终用户的环境中运行，因此只能访问给定最终用户有权访问的服务和资源。

这种区别在访问 [Firebase 实时数据库(RTDB)](https://firebase.google.com/docs/database/) 服务时最为明显。RTDB 支持指定[安全规则](https://firebase.google.com/docs/database/security/)，管理哪些用户被允许访问数据库的特定部分。以下面的规则配置为例:

清单 3:基于 UID 的访问控制的 Firebase 安全规则

该规则将对数据库节点`users/<uid>`的写访问权授予具有匹配用户 id 的认证用户。例如，用户`alice`被授权写给`users/alice`，但是`users/bob`对她是禁止的。一旦投入运行，RTDB 将根据此规则评估来自 Firebase 客户端 SDK 的所有写请求，并相应地批准或拒绝它们。但是，使用 Admin SDK 编写的应用程序可以不受限制地访问整个数据库。由于 Admin SDK 以更高的权限运行，并且它不是作为任何个人最终用户执行，因此安全规则不适用于 Admin SDK。这是为什么 Admin SDK 不应该部署在不受信任的环境中的主要原因之一。

然而，在某些情况下，这种不可知论者的行为是不可取的，甚至是完全危险的。在许多用例中，我们希望 Admin SDK 按照我们制定的规则运行。例如，想象这样一种情况，您依赖规则来维护保存数据的完整性。在这种情况下，给 Admin SDK 留下意外修改或删除数据库一部分的空间可能是灾难性的。幸运的是，我们可以很容易地将 Firebase Admin SDK 配置为[以有限的权限](https://firebase.google.com/docs/database/admin/start#authenticate-with-limited-privileges)操作，因此遵守了 RTDB 安全规则。这是通过在初始化 Admin SDK 时指定`databaseAuthVariableOverride`选项来完成的。

清单 4:用有限特权初始化管理 SDK(Java)

在上面的例子中，Admin SDK 进行的任何 RTDB 调用都将被视为由名为`my-service-worker`的经过身份验证的最终用户进行的，并且规则将照常执行。

## 后续步骤…

希望这有助于更清楚地描述 Firebase Admin SDKs 提供了什么，以及它们适合什么样的用例。更重要的是，我希望它能帮助您理解移动和 web 应用程序的服务器端集成的重要性，以及 Firebase 平台是如何促进这一点的。接下来我将推荐的是尝试用 Firebase Admin SDKs 实际构建一些东西。你可以从官方的[安装指南](https://firebase.google.com/docs/admin/setup)开始，挑选你喜欢的编程语言，然后继续尝试。还要注意，Firebase Admin SDKs 是[开源的](https://opensource.google.com/projects/firebase-sdk)。因此，请随时报告您遇到的任何问题，记录新功能请求，并随时提供建议和请求。