# 使用数据流将 RDBMS 复制到 BigQuery

> 原文：<https://medium.com/google-cloud/rdbms-replication-to-bigquery-using-datastream-658fd38b17ec?source=collection_archive---------0----------------------->

Google Cloud 在 9 月 22 日发布了预览模式下的一项新功能，让您可以使用 DataStream 将数据从 RDMBS(目前支持 Oracle，Postgress，MySQL)近乎实时地复制到 Big Query。BigQuery 使用变更数据捕获(CDC)功能和存储写入 API，以近乎实时的方式直接从源系统高效地复制更新。

![](img/9a2c3cb912a48836ab2cd14904f49477.png)

在这篇博客中，我们将展示设置此功能所需的所有步骤——为了演示，我们将在 MySQL 和 BQ 之间设置复制。我们开始吧

**步骤 1** —这是运行 MYSQL 服务器的先决条件，我已经在 GCE 上为这个博客设置了一个。

你可以点击这个[链接](https://phoenixnap.com/kb/how-to-create-a-table-in-mysql)在本地或者 GCE 上建立一个 MySQL 数据库。在这篇博客中，我创建了一个安装了 MySQL 服务器的 GCE 实例。还要为数据流创建一个新用户，以便能够连接到 MySQL

> 创建由“<password here="">”标识的用户“数据流”@“%”；</password>
> 
> 授予复制从属、选择、重新加载、复制客户端、锁定表、在*上执行。*到“数据流”@“%”；
> 
> 刷新权限；

创建一个测试表，我们将尝试在 BQ 中复制它

> 创建数据库测试；
> 
> 创建表 test _ rep(some _ misc _ value int)；
> 
> 如果存在，则删除过程；
> 
> 分隔符//
> 
> 创建过程 doWhile()
> 
> 开始
> 
> 声明 I INT DEFAULT 0；
> 
> WHILE(I<= 1000) DO
> 
> INSERT INTO test_rep (some_misc_value) values (i);
> 
> SET i = i+1;
> 
> END WHILE;
> 
> END;
> 
> //
> 
> CALL doWhile();
> 
> //

Datastream depends on CDC of source system to replicate the changes to BQ , we need to tweak a few config variables on the source side

Run the following command in your server command line

> sudo vi /etc/mysql/mysql.cnf

Paste the following lines in the file

> [mysqld]
> log-bin = MySQL-bin
> server-id = 1
> bin log _ format = ROW
> log-slave-updates = true
> expire _ logs _ days = 7

重新启动数据库服务器

登录你的 MySQL 数据库，确认这些配置是否已经设置好

> 显示全局变量，如“% binlog _ format %”；
> 
> 显示全局变量，如“expire _ logs _ days”；

显示的值应该与我们在上面的配置中设置的值同步。

**步骤 2** —在这一步中，我们为数据流创建一个连接配置文件，以便能够与 MySQL 服务器对话。转到数据流连接配置文件页面并选择 MySQL

![](img/ff84934f92498fc06115823a76cc17d6.png)

并给出源数据库服务器的连接细节

![](img/2c49b92adf80cd2f9778624a221d5d7e.png)

在“加密”部分，我们选择“无”来演示。对于连接选项，我们将选择“IP allowlisting ”,这相对于其他选项来说比较简单，您可以在此处探索其他选项[。](https://cloud.google.com/datastream/docs/network-connectivity-options?_ga=2.235514306.-2103651308.1662343754&_gac=1.116607476.1665672728.CjwKCAjw7p6aBhBiEiwA83fGunfLfhcWbdZfdLmQwP0XuTaFW5EOwgr_1DN1U5U9CwJ_weu3W4k1nBoCv-gQAvD_BwE)

![](img/0fbc25e3793dcf7c9a09296299814b7f.png)

请注意，在连接方法中，您将获得一组 IP，您的源数据库将从这些 IP 获得传入连接，请复制这些 IP，因为在步骤 3 中将需要它们

哇哦。我们几乎完成了连接配置文件，并准备测试连接，但在此之前，我们需要分开一点，回到我们的源 MySQL 服务器。

**步骤 3** —为了允许数据流连接到 MySQL 服务器，我们需要打开 MySQL 与之通信的端口 3306。

如果您正在使用 GCE 托管服务器，请创建一个新的入口防火墙规则，允许来自我们在步骤 2 中获得的 IP 的流量通过端口 3306

![](img/9b21b82d35696490af9c135545418344.png)

登录到您的 MySQL 服务器实例并运行以下命令

> sudo VI/etc/MySQL/MySQL . conf . d/mysqld . cn f

确保您的端口设置为 3306

> 端口= 3306

并将 bind-address 更改为 0.0.0.0(您也可以给出在步骤 2 中获得的逗号分隔的 IP 地址)

> 绑定地址= 0.0.0.0

这是我的配置文件的截图

![](img/a38f4a7eba107a98ecb019aa974d550e.png)

将更改提交到配置文件会让您的数据库服务器重新启动

> sudo systemctl 重启 mysql

运行以下命令以确保

> sudo ufw 允许从<ipaddress>到任何港口 3306</ipaddress>

我们完成了源系统的设置。

**步骤 4** —转到最后一节测试连接配置文件并运行测试

![](img/10bd7f382ea94400dcba7452239eee93.png)

现在，单击“创建”连接配置文件

如果您的连接测试失败，请重新阅读上面的说明，看看您是否遗漏了什么。

**第 5 步** —在这一步中，我们将设置目标 BQ 数据集和表，我们希望在其中设置复制同步

第六步——现在到了有趣的部分。创建数据流，这里面有 6 个子部分

6.1 —流名称和源/目标系统类型信息

![](img/d92b9d34497659506f361e07acb0141c.png)

6.2 —连接配置文件—在此选择在本博客的步骤 2 中创建的配置文件

![](img/e097edfa10456df4f1e1cdac6ed1364f.png)

6.3 —配置源流，在这里您可以选择我们想要复制的数据库和其中的表。我已经选择了我们在步骤 1 中创建的测试数据库和底层表

![](img/605e9bf4de96f289d1b1603d1a1afdae.png)

6.4 —输入目标连接配置文件的详细信息

![](img/349617c91d0c8cd7507ca1f2319941d6.png)

6.5 —在本节中，您将配置目标设置，您可以为每个模式选择不同的数据集，也可以将它们放在一个数据集下。我们会选择后者

![](img/d0bf177e9b048fb2bc391df443cbbb8e.png)

另请注意，我将过时限制保持为 0，因此这将为我们提供几乎接近时间的复制，您可以根据您的要求进行调整。请注意，这将对成本产生影响。

6.6 —全部完成，检查并创建！

在我们创建之前，点击底部的验证流选项

![](img/25a16421a2feeb3e651ae103b343c7cb.png)![](img/cb172b80bee584fd048ebc50995b2da1.png)

一旦创建了流，就启动流

![](img/57ff03e3b9aa08afdbdc3ea822556745.png)

**第 7 步** —检查目标同步(又名 BQ)数据集

如我们所见，已经创建了一个新表

![](img/12bd5ef7096b66b2849befb1f2f3b136.png)

查询该表

![](img/010a87d6bee95387a415db201672b18c.png)

我们可以在 Monitoring 部分看到关于我们新创建的数据流的各种统计信息

![](img/6630293f7f44b1d4c727b9079a65c884.png)![](img/4ba57b317888d1c7784434d73bd318c4.png)

全部完成！您可能需要清理并删除已创建的任何 GCP 对象，以避免不必要的开销。

**总之**在这篇博客中，我们建立了一个数据流，将数据从 MySQL 导入 BQ，这个过程非常简单，完全不需要服务器。我们可以使用类似的步骤来复制 Oracle 或 PostgreSQL，只需稍作修改。