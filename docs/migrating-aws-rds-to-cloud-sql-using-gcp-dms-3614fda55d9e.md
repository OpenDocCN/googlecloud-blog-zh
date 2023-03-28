# 使用 GCP DMS 将 AWS RDS 迁移到云 SQL

> 原文：<https://medium.com/google-cloud/migrating-aws-rds-to-cloud-sql-using-gcp-dms-3614fda55d9e?source=collection_archive---------0----------------------->

![](img/c19c1ce42f405d577bc1eb02304cb9db.png)

[Growtika 开发商营销机构](https://unsplash.com/@growtika?utm_source=medium&utm_medium=referral)在 [Unsplash](https://unsplash.com?utm_source=medium&utm_medium=referral) 上拍摄的照片

如果您有在 AWS 上运行的任务关键型工作负载，您可能正在寻找一种方法来迁移到 GCP，同时保持相同的数据库引擎，将停机时间降至最低或接近零。本博客提供了使用 [**GCP 数据迁移服务将您的 AWS RDS 迁移到 GCP 云 SQL 的解决方案。**](https://cloud.google.com/database-migration)

GCP DMS: GCP 数据库迁移服务(AWS DMS)是一项托管迁移和复制服务，可帮助您快速、安全地将数据库和分析工作负载迁移到 GCP，同时最大限度地减少停机时间，实现零数据丢失。GCP DMS 支持各种数据库和分析引擎的迁移，如 MySQL 和 PostgreSQL，并在 preview 中支持 SQL Server 和 Oracle 迁移。

这篇博文将通过建立 VPN 连接，使用**数据迁移服务**将数据库从 Amazon RDS MySQL 实例迁移到 Google Cloud SQL 迁移实例。

## 先决条件

在开始之前，您必须完成以下先决条件:

*   AWS 和 GCP 控制台访问

## 术语:

以下是本指南中使用的术语定义。

**Cloud SQL:** 一个完全托管的关系数据库服务，它使**易于设置和** **管理**您在云中的 **PostgreSQL、MySQL、& SQL Server** 数据库。

**云 DMS:** 轻松将您的 AWS RDS 数据库提升并迁移到云 SQL，最大限度减少停机时间。

迁移您的整个数据库，包括数据和元数据，无需对源进行任何额外的更改。

DMS 是一项免费服务，允许你只为使用你全新的 GCP 数据库付费。

**IP allowlist:** 当源数据库在 Google Cloud 外部，并且具有外部可访问的 IPv4 地址和 TCP 端口时，公共 IP 连接是最合适的。如果源数据库托管在 Google Cloud 的另一个 VPC 中，那么连接源数据库和云 SQL 实例的最简单方法就是使用 VPC 对等。

**亚马逊 RDS** :亚马逊关系数据库服务是亚马逊 Web Services 提供的分布式关系数据库服务。它是一个运行在“云中”的 web 服务，旨在简化应用程序中使用的关系数据库的设置、操作和伸缩。

## 对于 AWS 站点到站点 VPN，请遵循以下步骤:

步骤 1:创建一个 VPC

名称:来源-vpc，地区:美国东部-1，VPC CIDR: 192.168.17.0/24，

![](img/8396d4ead6a811b6d96ff598354b78a7.png)

步骤 2:在上述 RDS VPC 的不同可用性区域中创建两个子网:

子网 1: 192.168.17.127/28，子网 2: 192.168.17.144/28

![](img/9cd4662f4d325ee90b9368b262f636e3.png)![](img/52cd7bea5ca058fe3df9354db789a8d5.png)

步骤 3:使用 34.23.85.116 的 GCP 经典 VPN 网关 IP(**)创建一个客户网关**

**名称:aws-cg，ip 地址: **34.23.85.116(** GCP 保留静态公共 IP)**

**![](img/65a5ab2948769b8a80964f20d3a2b517.png)**

**步骤 4:创建虚拟专用目标网关:**

**名称:aws-vpg**

**![](img/7b7088769f5e7e9c1fb9a3741315a752.png)**

**步骤 5:将虚拟专用网关连接到 VPC**

**![](img/31d4455f681bcff473b3cd2216023ffd.png)**

**步骤 6:创建站点到站点 VPN 连接**

**名称:aws-gcp-vpn，虚拟专用网关:aws-vpg，客户网关:aws-cg，路由选项:静态**

**本地 IPv4 网络 CIDR: 192.168.17.0/24(AWS VPC IP)**

**远程 IPv4 网络 CIDR:192.168.16.0/24 (GCP VPC IP)**

**![](img/2217b73984abd875126c58ec6ca01d38.png)****![](img/fcaedd95c445af22c5b7ff75710149cd.png)**

**步骤 7:下载配置文件(复制 GCP VPN 隧道设置所需的**预共享**密钥)**

**![](img/589706b713ab76c804aacbf687bb77d9.png)**

**预共享密钥:**jfS _ . xpb 6 rad 4v vud . 5 jq 6 kou u9 ybv 4 l****

**![](img/a0ba18f9614211387c3fbdd80ec07120.png)**

**步骤 8:隧道状态当前为关闭，因为我们尚未在目标端完成隧道设置**

**![](img/36551db5e33719073dc988761218bd14.png)**

**步骤 9:添加入站安全规则，以允许来自云 VPN (192.168.16.0/24)的所有协议和端口。**

**![](img/94efcfc9bf5741552c61d24fb1d56a4a.png)**

**将两个子网都关联到 VPC 主路由表:**

**VPC →路由表→编辑子网关联**

**![](img/afbff6761fcd5bb9c5c89d3407b2881e.png)**

**更新路由传播:**

**VPC →路由表→编辑路由传播**

**![](img/bd46c1a1375887627e748c9cc6ac9dfe.png)**

**更新 GCP CIDR 山脉的路线:**

**VPC →路由表→编辑路由**

**![](img/6e9684bc5f215e866c78e0d1b78ddb12.png)**

**步骤 10:配置客户网关设备(GCP VPN)**

## **对于 GCP 经典 VPN，请遵循以下步骤:**

> **步骤 1:在 GCP 控制台中启用以下 API:**

**计算引擎 API**

**![](img/62bd77c50abbeb5d1e2aca0e31936df4.png)**

**服务网络 API**

**![](img/089f6f5d3c249d71003eede6229d6eee.png)**

> **步骤 2:创建目标 VPC**

**名称:塔吉特-vpc，VPC CIDR: 192.168.16.0/24，地区:美国东部 1**

**![](img/553d0f751fcbc8a5d240f309d7f39816.png)****![](img/e870f7090cb81793f61eb8f9b86cede3.png)**

> **步骤 3:为传统 VPN 网关保留一个公共 IP**

**名称:静态-公共-IP-经典-vpn**

**![](img/ad6993c93b0ae33c1171d216b715f1aa.png)**

> **步骤 4:创建一个经典的 VPN 网关**

**![](img/39f64370129e0b16fc5b00963734b25a.png)**

**名称:gcp-vpn-gw，网络:target-vpc，地区:us-east1，ip 地址:保留的公共 IP**

**![](img/e63a34f17a3d92952035f689c468946a.png)**

> **步骤 5:设置隧道**

**名称:gcp-aws-vpn-tunnel，远程对等 ip 地址:3.230.94.148，IKE 版本:IKEv1，**

**IKE 预共享密钥:jfS _ . xpb 6 rad 4v vud . 5 jq 6 kou u9 ybv 4 l，**

**路由选项:基于路由的远程网络 IP 范围:192.168.17.0/24**

**![](img/1f1354a38029cb1ac5ade1a63129b43a.png)**

**第六步:**

**![](img/760c32ce9aaf27231d220c5662cb7b13.png)**

**更新 AWS 流量的 GCP 防火墙规则:**

**VPC 网络→防火墙→添加防火墙规则**

**![](img/e92f290e4ddcea895df013b2a26442cc.png)**

**添加防火墙规则:**

**![](img/25844e4dd3ce8fbffac658ad22c10656.png)**

## **验证:**

**AWS 端隧道状态应该为运行:**

**![](img/a69669ace8c2e05314b461494f3a2e38.png)**

**应建立 GCP VPN 隧道状态:**

**![](img/0746633c1571182c505fc4d3a168f5f4.png)**

## **按照以下步骤创建 AWS RDS 实例:**

1.  **数据库创建方法**

**![](img/c717d80fb98954d46261b69fdc2adaeb.png)**

**2.模板**

**![](img/dd50e712e045688c92dfa8175918f088.png)**

**3.连通性:**

**![](img/f9a9f6a7499fbb2dfa52b01ea0f1c8d1.png)**

**4.数据库认证:**

**![](img/0d25049330bfc6b5d4a25671d1ad8532.png)**

**5.复制 RDS 实例端点:**

**终点:aws-rds.ciukybaxmgck.us-east-1.rds.amazonaws.com**

**6.将数据转储到 RDS。**

## **按照以下步骤设置 DMS 并启动迁移作业:**

1.  **登录 AWS 并配置您的[源数据库](https://cloud.google.com/database-migration/docs/mysql/configure-source-database)。**
2.  **启用数据迁移 API:**

**![](img/46bed6e9fd6b652d16896d0289d822ab.png)**

**3.在 GCP 创建一个连接配置文件:在谷歌云导航菜单中，转到数据迁移服务。在 DMS 中，单击连接配置文件并创建一个新的连接配置文件。**

****注意:**连接配置文件用于存储源数据库信息。使用连接配置文件，我们将连接到源并创建我们的迁移作业。连接配置文件可重复用于多个迁移作业。**

*   **源数据库引擎— Amazon RDS for MYSQL**
*   **主机名—来自 Amazon RDS MYSQL 实例的端点**

**此菜单将创建一个名为“aws-rds-cloud-sql-connection”的新连接配置文件**

**![](img/2e3a4092ea6030b0100784343b71dc92.png)**

**4.创建连续/一次性迁移作业**

**创建新的迁移作业 **→** 定义源实例→定义目标→定义连接方法→**

**" AWS-rds-cloud-SQL-迁移-作业"**

**![](img/1504bbcf3e01cd94447edd7dfc9071ee.png)**

**5.定义来源:**

**![](img/753c1523a54b0d411b3a46be7f4200d1.png)**

**6.创建目的地:**

**![](img/c8c9903aeb0439917bf41486ceb5da75.png)****![](img/ea81fef7cd5494cdd5be8e4428782afa.png)****![](img/479e9eef4341f81b3223871785222726.png)**

**7.定义连接方法:**

**![](img/1db856af99dc28a6385c5f626cf1b2f7.png)****![](img/d60636a96e8e029f5fc013d7ddf79b3c.png)**

**8.测试迁移作业，它将失败，因为我们尚未配置防火墙和路由:**

**![](img/1bea2da87af65dd908af4a3a64e2f84f.png)**

# **允许来自 AWS 和 GCP 两端的防火墙用于云 SQL:**

## **在 GCP 侧执行以下步骤:**

**编辑该 VPC 对等，在 VPC 对等连接详情中勾选`Import Custom Routes`和`Export Custom Routes`，点击**保存**。**

**![](img/dd28f02230c1ba8f01d3ded5d66d7c12.png)**

**复制 google 托管云 SQL 的内部 ip 范围:**

**云 SQL 的内部 ip 范围:10.241.128.0/20(需要此 IP 范围才能在 AWS 安全组中允许)**

**![](img/91ea5762d72d5d0581e610c432a8eb09.png)**

**为专用连接启用自定义路由:**

**在控制台中搜索 VPC→专用服务连接→专用服务连接→导出自定义路由**

**![](img/592a5210b6aaafe70c5f926d9859ae16.png)****![](img/e92f290e4ddcea895df013b2a26442cc.png)**

**9.创建并开始作业**

**![](img/753c1523a54b0d411b3a46be7f4200d1.png)**

## **在 AWS 端执行以下步骤:**

**验证路线:**

**更新云 SQL 的静态路由:**

**站点到站点 VPN 连接→静态路由→编辑路由**

**![](img/cca3bf250c5af0b4ef019883f70cbad9.png)**

**更新云 SQL 内部 IP 范围的 AWS 安全组(10.241.128.0/20):**

**VPC →安全组→编辑入站规则**

**![](img/68ed0dc335e193ef6da9dbced99a38f6.png)**

**8.对防火墙进行更改后，再次测试迁移作业:**

**![](img/fad043f1af2b3bd353e2e4819beb3b90.png)**

**10.在源数据库(AWS RDS)中将 [BINLOG_FORMAT](https://cloud.google.com/database-migration/docs/diagnose-issues?hl=en_US&_ga=2.190913322.-1273888051.1671891078) 从 Mixed 更改为 Row。**

**![](img/b113266815e4453d8325bb5c025245a7.png)**

**11.对源数据库进行更改后，再次测试迁移作业:**

**![](img/3856c047dc48e0e0c167fce692f4c23e.png)****![](img/0b90672c712bdc329d083597203ccaec.png)**

**提升数据库:**

**![](img/e202ecbbca2b24127ea12301e9d17574.png)****![](img/2e43dd394125b5b7a687f9a1f97cc95f.png)**

## **验证:**

**登录云 SQL 并验证数据:**

**![](img/882da97a4bb38ac1278c9fbb95b8ba6b.png)**

**参考链接:**

**[](https://cloud.google.com/database-migration/docs/mysql/configure-connectivity-vpns) [## 使用 VPN 配置连接|数据库迁移服务|谷歌云

### 如果您的源数据库在 VPN 中(例如，在 AWS 中，或者您的内部 VPN 中)，您还需要在…上使用 VPN

cloud.google.com](https://cloud.google.com/database-migration/docs/mysql/configure-connectivity-vpns) [](https://cloud.google.com/database-migration/docs/mysql/configure-source-database) [## 配置您的源|数据库迁移服务|谷歌云

### 无论您的企业正处于数字化转型的早期阶段，谷歌云都可以帮助您解决…

cloud.google.com](https://cloud.google.com/database-migration/docs/mysql/configure-source-database)**