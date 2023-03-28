# SFTP 访问谷歌云存储

> 原文：<https://medium.com/google-cloud/sftp-access-to-google-cloud-storage-43ffd6134b0e?source=collection_archive---------0----------------------->

![](img/e02835eb7be15d6da0dba38827a724cf.png)

2022–07:自从第一次发表这篇文章以来，在用于为 SFTP 提供 GCS 访问的开源领域出现了一个新的参与者。这是一个名为 [SFTPGo](https://github.com/drakkan/sftpgo) 的项目。另外一篇[文章](/@kolban1/sftpgo-access-to-gcs-via-sftp-e203e0783f6f)已经写好了关于结合 GCS 使用 SFTPGo。

谷歌云存储(GCS)为数据提供 blob 存储。文件可以上传到 GCS，随后进行检索。这种存储价格低廉，并且具有出色的可用性和耐用性。GCS 提供了各种编程语言 API，可由定制应用程序使用，并且 Google 的许多产品都是预先构建的，用于向 GCS 生成数据和从 GCS 消费数据。命令行工具如`gsutil`也提供脚本访问。如果数据可以使用存储传输产品通过 web 寻址，则可以自动接收数据。

Google 在开箱即用的 GCS 故事中提供的*而不是*是通过任何文件传输协议(FTP)访问 GCS 数据的能力。在本文中，我们描述了通过[安全文件传输协议](https://en.wikipedia.org/wiki/SSH_File_Transfer_Protocol) (SFTP)和相应的 SFTP 客户端工具访问 GCS。

SFTP 是一种开放式规范，用于与远程文件系统交互，以在分层文件系统中存储、检索和列出文件。SFTP 交换所有数据，并对飞行中的流量进行完全加密。这意味着无论你在移动什么数据，都不能通过网络进行检查。SFTP 常见的公共客户包括:

*   [sftp](https://linux.die.net/man/1/sftp) (Linux/Unix)
*   [WinSCP](https://winscp.net/eng/index.php) (Windows SFTP 客户端)
*   [FileZilla](https://filezilla-project.org/)
*   [赛博鸭](https://cyberduck.io/)
*   [其他](https://en.wikipedia.org/wiki/Category:SFTP_clients) …

SFTP 规范有一些优秀的开源库实现，这意味着我们可以编写客户端和服务器 SFTP 实现。其中一个库叫做 [ssh2](https://www.npmjs.com/package/ssh2) ，可用于 Node.js。使用这个库，我们编写了一个示例，它将自己公开为 SFTP 服务器，但使用 GCS 作为后端存储系统。这意味着 SFTP 客户端可以连接到我们的 SFTP 服务器(我们称之为 sftp-gcs)。然后根据 GCS 执行上传文件、获取文件、列出文件和其他文件操作的请求。从 SFTP 客户端的角度来看，它的行为与处理任何其他分层文件系统是一样的，区别在于数据是由 GCS 支持的。如果您的应用程序或工具目前希望使用 SFTP 处理文件数据，那么这可能是将 GCS 引入您的故事的一个极好的组件。

sftp-gcs 程序通过 [Github](https://github.com/kolban-google/sftp-gcs) 以源代码形式提供。要使用，您需要克隆存储库并运行应用程序。因为它是用 Node.js 在 JavaScript 中实现的，所以您还需要安装 [Node.js](https://nodejs.org/en/) 。安装完成后，我们使用以下工具运行应用程序:

```
node sftp-gcs.js --bucket=my-bucket
```

我们可以使用一些命令行标志。仅需要`bucket`。

*   `--bucket [BUCKET_NAME]` —要操作的桶的名称。
*   `--port [PORT_NUMBER]` —服务器监听的 TCP/IP 端口号。默认为 22。
*   `--user [USER_NAME]` —用户名，如果我们希望使用用户名登录。
*   `--password [PASSWORD]` —如果我们希望使用用户名登录，则输入密码。
*   `--public-key-file [PUBLIC_KEY_FILE]` —包含用于 SSH 登录的公共 SSH 密钥的文件。
*   `--service-account-key-file [KEY_FILE]` —包含服务帐户密钥的本地文件的路径。
*   `--debug [DEBUG_LEVEL]` —开启调试。为最大调试提供“调试”。

SFTP 服务器将充当 GCS 存储的网关。它被设计为每个实例只允许访问一个存储桶。我们总是可以配置多个实例，其中每个实例可以被定义为使用不同的存储桶。如果对多存储桶支持有需求/兴趣，可以在以后添加。

作为 SFTP 服务器，SFTP 客户端必须连接到它。这意味着它必须侦听 TCP/IP 端口。我们可以向端口提供`--port`参数。如果没有提供，默认为 22，这是 SSH 使用的相同端口。如果您计划使用 SSH 同时运行服务器，那么您可能需要提供一个备用端口。

安全性是首要考虑的因素，我们必须检查安全性的两个方面。首先是可以使用什么身份来访问 SFTP 服务器。第二个问题是，一旦 SFTP 服务器被访问，向 GCS 出示什么身份以允许访问？

让我们看看 SFTP 服务器访问。有三种可能性:

1.  没有安全感。对 SFTP 服务器的连接请求将立即成功，没有任何挑战。通过授予 SFTP 服务器的 GCS 权限，安全性将回到到 GCS 的连接。
2.  用户标识/密码。对 SFTP 服务器的连接请求将导致对用户 id/密码对的请求，该用户 id/密码对必须与配置期间提供的相匹配。
3.  SSH 键。只有当调用者拥有与配置给服务器的公钥相对应的私钥时，到 SFTP 服务器的连接请求才会成功。

要使用用户标识/密码安全，请提供`--user`和`--password`。

要使用基于 SSH 密钥的安全性，使用`--public-key-file`提供一个公钥文件。

如果不使用任何安全性，请不要提供`--user`、`--password`和`--public-key-file`中的任何一个。

一旦客户端连接到 SFTP 服务器，上传和获取文件的请求将被发送到 GCS。发出请求的身份要么是 Google 应用程序的默认凭证，要么是用`--service-account-key-file`指定的服务帐户。应用默认凭据是在 GCP 计算引擎中运行或设置环境变量`GOOGLE_APPLICATION_CREDENTIALS`时发现的隐式凭据环境。

一旦它开始运行，我们就可以将一个符合协议的 SFTP 客户端连接到服务器，并开始发出 SFTP 命令。以下视频说明了服务器的安装和使用。

# 问题和答案

**问**:我们可以在无服务器环境下运行这个解决方案吗，比如云功能或者云运行？

**答**:很遗憾不能。原因是，截至 2020 年 12 月，云功能和云运行都不支持除 HTTP(s)之外的任何协议。只能通过 Internet 接收 HTTP 请求进行处理。我们的 SFTP 解决方案使用 SSH 协议承载 SFTP 子协议请求。虽然我们可以让我们的 SFTP-GCS 服务器监听一个通常用于 HTTP(s)的 TCP 端口，但这没有用。除非我们能够从无服务器请求中发现 TCP 协议，否则我们不会取得任何进展。查看与计算引擎相关联的托管实例组(MIG)可能会有一些好处，但我不确定这些好处是否可以减少到零。

**问**:我如何在开机时启动恶魔？

**答**:在虚拟机启动时启动恶魔与其说是一个与*这个*恶魔相关的具体问题，不如说是一个一般性的问题。我的建议是研究一下`systemd`文档，这是 Linux 环境中引导管理的最新故事。有很多关于为`systemd`创建*单元*的好文章。

# 链接

*   [Github sftp-gcs](https://github.com/kolban-google/sftp-gcs)
*   [中:计划镜像/同步 SFTP 到 GCS—2019–03](/google-cloud/scheduled-mirror-sync-sftp-to-gcs-b167d0eb487a)
*   [文件图像 SFTP/FTP 网关](https://console.cloud.google.com/marketplace/product/filemage-public/filemage-gateway-linux)
*   [Trillo —云存储的文件管理器和 SFTP](https://console.cloud.google.com/marketplace/details/trillo-vm-prod/trillo-platform-vm)
*   [Couchdrop](https://couchdrop.io/)