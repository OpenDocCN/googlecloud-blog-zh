# MySQL 到云扳手:使用 Vertex AI 笔记本和 GCP Dataproc 无服务器将 MySQL 数据库迁移到云扳手

> 原文：<https://medium.com/google-cloud/mysql-to-cloud-spanner-migrate-mysql-database-to-cloud-spanner-using-vertex-ai-notebooks-and-gcp-ad7d2ed8a317?source=collection_archive---------5----------------------->

![](img/1d515877ea1769f253f9a0e3b92d82ca.png)

寻找将 MySQL 表并行迁移到 Cloud Spanner 的预构建解决方案？别再看了。我们开发了一个 Vertex AI 笔记本(Jupyter notebook)解决方案，它使用 dataproc 无服务器进行迁移。你只需要提供必要的参数，笔记本会帮你把一个完整的 MySQL 数据库迁移到 Cloud Spanner。

[Dataproc 模板](https://github.com/GoogleCloudPlatform/dataproc-templates)使用 [VertexAI 笔记本](https://cloud.google.com/vertex-ai/docs/tutorials/jupyter-notebooks)和 [Dataproc 无服务器](https://cloud.google.com/dataproc-serverless/docs)提供一站式解决方案，将数据直接从 MySQl 迁移到 Cloud Spanner。

1.从 [API 控制台](https://console.developers.google.com/)启用 GCP 项目中的以下服务:

*   计算引擎 API
*   Dataproc API
*   顶点人工智能应用编程接口
*   顶点笔记本 API

2.在[顶点 AI 工作台中创建一个](https://pantheon.corp.google.com/vertex-ai/workbench/list/instances)[用户管理的](https://cloud.google.com/vertex-ai/docs/workbench/user-managed/introduction)笔记本。

![](img/0a8fa76d9268f3e3e6e7dcfdaacd3278.png)

3.使用 GIT 选项卡克隆 Dataproc 模板 GitHub repo，如下图所示

![](img/f0e77d244e283de46973971dc8292220.png)

或者从启动窗口打开一个终端，使用 git clone 进行克隆。

`git clone [https://github.com/GoogleCloudPlatform/dataproc-templates.git](https://github.com/GoogleCloudPlatform/dataproc-templates.git)`

4.从文件夹选项卡中，打开路径为 data proc-templates/notebooks/MySQL 2 spanner 的[MySqlToSpanner _ notebook . ipynb](https://github.com/GoogleCloudPlatform/dataproc-templates/blob/main/notebooks/mysql2spanner/MySqlToSpanner_notebook.ipynb)笔记本

5.笔记本包含一步一步的过程，以帮助从 MySQL 迁移数据到云扳手。下面是迁移的步骤:

**第一步:安装所需的软件包**

迁移所需的一些软件包需要单独安装，因为它们在笔记本中不可用，例如 pymysql SQLAlchemy、JDK 等。

**步骤 2:导入库:**导入所需的包

**第三步:分配参数:**运行笔记本前需要设置以下配置:

**步骤 3.1:通用参数**

*   `PROJECT` : GCP 项目标识
*   `REGION` : GCP 地区
*   `GCS_STAGING_LOCATION` : GCS 暂存位置，用于此笔记本存储工件
*   `SUBNET` : VPC 子网
*   `JARS`:罐子列表。对于这个笔记本，除了 dataproc 模板 jar 之外，还需要 mysql 连接器和 avro jar
*   `MAX_PARALLELISM`:并行运行的作业数量参数，默认值为 2

**步骤 3.2: MYSQL 到扳手参数**

*   `MYSQL_HOST` : MYSQL 实例 ip 地址
*   `MYSQL_PORT` : MySQL 实例端口
*   `MYSQL_USERNAME` : MYSQL 用户名
*   `MYSQL_PASSWORD` : MYSQL 密码
*   `MYSQL_DATABASE`:要迁移的数据库的名称
*   `MYSQLTABLE_LIST`:您要迁移的表的列表，例如:['table1 '，' table2']否则提供一个空列表以迁移整个数据库，例如:[]
*   `MYSQL_OUTPUT_GCS_LOCATION`:将 mysql 输出写入的 gcs 位置，例如:“gs://bucket/[folder]”
*   `MYSQL_OUTPUT_GCS_MODE`:MYSQL 数据输出模式之一(覆盖|追加)
*   `MYSQL_OUTPUT_GCS_FORMAT`:MYSQL 数据的输出文件格式为(avro|parquet|orc)之一
*   `SPANNER_INSTANCE`:云扳手实例名
*   `SPANNER_DATABASE`:云扳手数据库名称
*   `SPANNER_TABLE_PRIMARY_KEYS`:为 MYSQL 中没有主键的表提供格式为{"table_name":"primary_key"}的字典

**步骤 3.3:笔记本配置参数**

这些参数包含 Dataproc 作业所需的 jar 和属性的路径。除非您想要更改 jar 的位置，否则不需要更改这些参数。

**第四步:生成 MySQL 表列表**

这一步生成要迁移的表列表。如果步骤 3.2 中的`MYSQLTABLE_LIST`参数为空列表，则选择数据库中的所有表格。

**步骤 5:获取 SPANNER_TABLE_PRIMARY_KEYS 中不存在的表的主键**

该步骤获取出现在`MYSQLTABLE_LIST`参数中的表的主键，如果这些表的主键未在步骤 3.2 的参数`SPANNER_TABLE_PRIMARY_KEYS`中提供

**步骤 6:获取表的行数并识别读分区列**

该步骤使用`PARTITION_THRESHOLD`(默认值为 200k)参数，任何行数大于`PARTITION_THRESHOLD`的表都将基于主键进行分区

**第七步:计算 MySQL 到 Cloud Spanner 的并行作业**

基于步骤 3.1 中设置的`MAX_PARALLELISM`参数，该步骤计算 MySQL 到 GCS 迁移和 GCS 到 Cloud Spanner 的并行作业。基本上是将并行迁移到 Cloud Spanner 的 MySQL 表的数量。

**步骤 8:创建 JAR 文件并上传到 GCS**

该步骤创建所需的 JAR 文件，并上传到步骤 3.1 中定义的`GCS_STAGING_LOCATION`

**步骤 9:执行管道将表从 MySQL 迁移到 Spanner**

这一步启动 MySQL 表迁移的顶点 AI 管道执行。为每个作业生成管道链接。我们还可以在 Dataproc UI 的[批处理](https://console.cloud.google.com/dataproc/batches?_ga=2.256091016.2085864710.1666708114-43261181.1657894175)部分看到作业正在运行

![](img/a27f962c907000b9e911d43424e02209.png)![](img/b62df8e310eb0449a31b1279baa15e7d.png)

**步骤 10:获取从 MySql 迁移到 Spanner 的表的状态**

该步骤获取在步骤 9 中执行的作业的状态

![](img/cae60f73d669b5d74c2133220ba442ce.png)

**步骤 11:验证从 MySQL 迁移到 Cloud Spanner 的表的行数**

这些步骤有助于验证从 MySQL 迁移到 Cloud Spanner 的每个表的记录数量

![](img/a246c52f7389c3dfe7d4247d8ac32682.png)

如有任何疑问/建议，请联系:dataproc-templates-support-external@googlegroups.com