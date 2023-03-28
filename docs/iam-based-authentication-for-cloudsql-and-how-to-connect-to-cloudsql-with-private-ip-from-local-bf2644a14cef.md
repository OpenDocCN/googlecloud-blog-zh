# 基于 IAM 的 CloudSql 认证以及如何从本地机器使用私有 ip 连接到 CloudSql。

> 原文：<https://medium.com/google-cloud/iam-based-authentication-for-cloudsql-and-how-to-connect-to-cloudsql-with-private-ip-from-local-bf2644a14cef?source=collection_archive---------1----------------------->

Google Cloud CloudSQL 原生支持 IAM 集成。因此，用户可以使用他们的电子邮件云身份登录到云 SQL，并且可以管理用户对数据库的权限。此过程使用短期身份验证令牌，不需要密码。日志中会跟踪用户的登录活动。

对您的所有数据库实施 Cloud SQL IAM 数据库身份验证使数据库的用户管理更加安全、高效。

让我们讨论如何在云 Sql 中设置 IAM 数据库认证。

第一步是在 CloudSql 实例上启用基于 IAM 的身份验证，下面是可以用来实现这一点的命令

cloudsql.iam_authentication=on

从控制台，我们也可以通过设置标志来启用它

IAM 身份验证允许用户使用他们的云身份连接到 CloudSql。任何数据库级别的授权都需要在数据库本身中处理。

即授予对表、模式、数据库的读访问或写访问权限，仍然需要作为数据库命令而不是 IAM 角色来授予。

一旦 IAM 身份验证标志设置为 on，下一步就是向用户或组授予“云 SQL 实例用户”角色。该角色允许用户使用他们的云身份认证到 cloudsql。

授予角色后，下一步是将用户或组添加到 CloudSql 实例，如下所示，

导航到“控制台→ SQL →选择实例→用户→添加用户帐户→云 IAM →输入电子邮件地址→添加”

此时，用户可以连接到 CloudSql 并登录数据库。尽管如此，这并不能授予用户查询表或模式所需的访问权限。这仅包括认证部分，授权访问需要在数据库端被授予。

让我们看看如何创建一个角色，并将其授予云身份或组。

创建角色 role _ ro

\ connect databaseName

将数据库 databaseName 上的 CONNECT 授予 role _ ro

向 role_ro 授予对架构 schemaName 的使用权；

将 SCHEMA schemaName 中所有表上的 SELECT 权限授予 role _ ro

创建角色后，下一步是将角色授予用户或组。

将 role_ro 授予[user@example.com](mailto:user@example.com)

**连接到云 SQL 使用** g **IAM 数据库认证**

连接到云 SQL 实例的安全方式是通过 [CloudSQL Auth proxy](https://cloud.google.com/sql/docs/postgres/sql-proxy) 或使用 Java/Python Native client。

在这篇博客中，我们将讨论如何使用 Auth proxy 连接到具有公共和私有 ip 的 CloudSQL 实例。

首先，我们需要安装最新版本的 Cloud SQL auth 代理二进制文件，然后认证到 Google Cloud IAM。

$ gcloud 授权登录

登录成功后，让启动代理连接

*   具有公共 IP 的实例

。/cloud _ SQL _ proxy-enable _ iam _ log in-instances = PROJECT:REGION:cloud SQL _ INSTANCE _ NAME = TCP:port

一旦连接建立，现在我们可以使用云身份连接到数据库。

psql " host = 127 . 0 . 0 . 1 port = $ CLOUDSQLPROXYPORT DBNAME = $ DBNAME user=user@example.com SSL mode = disable "

上面的命令从下面的环境变量中获取凭据。

GOOGLE _ 应用程序 _ 凭据

因为我们是公开连接到 CloudSQL 实例，所以它需要一个 SSL 证书——因此，slmode 参数被设置为 disable。但是，云 SQL Auth 代理确实提供了加密连接。

*   具有私有 IP 的实例

如果您有到 gcp 网络的 VPN 连接或到 GCP 网络的专用互连，上述方法对于 privateIp 实例同样有效。

如果您没有 vpn 或互联网连接，则无法从您的机器启动代理并连接到 CloudSql。有一个仅用于测试目的的解决方法，即在与 CloudSql 实例相同的网络中创建一个虚拟机，并使用 CloudIAP 隧道连接到该虚拟机。从虚拟机上，我们可以启动 Cloudproxy。同样，这是一个权宜之计，应该只用于开发环境。

对于 TCP 连接，GCP 的[身份识别代理(IAP)](https://cloud.google.com/iap) 支持通过私有 IP 从互联网访问 GCE 虚拟机。当试图建立到代理的 HTTPS 加密隧道时，代理执行认证和授权检查。在成功的认证和授权之后，来自客户端的流量通过代理在 Google 的内部网络上被转发到 VM 实例。换句话说，代理充当对外界开放的传入流量的看门人，而 VM 只需要允许来自 Google 内部网络上的代理的连接。

隧道配置

为了配置 IAP 通道，我假设您有以下条件

*   您拥有的 GCP 项目，带有正在运行的 GCE 虚拟机，对于 Windows 虚拟机，登录和密码—确保在创建虚拟机时禁用公共 ip 的分配—对于 Windows 节点，确保在创建虚拟机后不要忘记指定用户名和密码
*   安装在您想要连接的系统上的 [Google Cloud SDK](https://cloud.google.com/sdk)
*   如果你是公司代理，你可能需要[将 IAP 用于 TC 的域名“tunnel.cloudproxy.app”列入白名单](https://cloud.google.com/iap/docs/faq#what_domain_does_for_tcp_use)

让我们首先创建一个防火墙规则，允许流量从 IAP 流向 VM。IAP 使用范围 35.235.240.0/20 作为转发流量的源地址。我们允许端口 22 (ssh)。

g cloud compute firewall-rules create allow-INGRESS-from-IAP \-direction = INGRESS \-action = allow-rules = TCP:3389，tcp:22，TCP:5901 \-source-ranges = 35 . 235 . 240 . 0/20

上面的命令允许从代理到项目中所有节点的连接。如果您不想为项目中的所有虚拟机打开防火墙，您可以[将标签](https://cloud.google.com/sdk/gcloud/reference/compute/instances/add-tags)附加到您希望应用规则的虚拟机，并使用'— target-tags=TAG '作为附加选项指定该标签。

现在，用户需要被授予 IAM 策略，以允许他们建立隧道连接。为此，我们需要向授权用户和/或用户组授予 iap.tunnelResourceAccessor 角色。在这个例子中，这个角色是在项目级别设置的。这允许用户建立到项目中应用上述防火墙规则的所有虚拟机的连接。

gcloud 项目添加-iam-策略绑定 IAP-访问-测试\

—member=user:user@example.com

—role = roles/IAP . tunnelresourceactor

用户还需要拥有 compute.viewer 角色

gcloud 项目添加-iam-策略绑定 IAP-访问-测试\

—member=user:user@example.com

—角色=角色/计算.查看器

**连接 SSH:Linux-GCE 上的 Linux**

如果您想通过 ssh 上的端口转发连接到 Linux 节点，请使用下面的命令。

g cloud compute ssh IAP-test-Ubuntu—项目 iap-access-test \

—区域 us-central 1-a—ssh-flag "-L 5901:localhost:5901 "

现在，您将能够 ssh 到与 CloudSql 实例位于同一个 VPC 网络中的 VM，您可以按照上一节中的 cloudSql 代理设置进行操作，并且能够在不登录 VM 的情况下测试您的连接和 db 设置。