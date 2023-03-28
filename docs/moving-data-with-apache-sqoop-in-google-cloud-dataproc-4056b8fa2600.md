# 在 Google Cloud Dataproc 中使用 Apache Sqoop 移动数据

> 原文：<https://medium.com/google-cloud/moving-data-with-apache-sqoop-in-google-cloud-dataproc-4056b8fa2600?source=collection_archive---------0----------------------->

![](img/434b648d1035f5cbc50f8a70dcc83d3a.png)

有关系数据库吗？—想用 [Apache Sqoop](https://sqoop.apache.org/) 把这个数据库导出到[云存储](https://cloud.google.com/storage/)、 [BigQuery](https://cloud.google.com/bigquery/) ，或者 [Apache Hive](https://hive.apache.org/) ？—想通过 [Cloud Dataproc](https://cloud.google.com/dataproc/) 来完成这一切，这样您就只需为您使用的东西付费了？

希望你回答**是**因为这就是这篇文章的内容！

Cloud Dataproc 非常棒，因为它可以快速创建一个 Hadoop 集群，然后您可以使用它来运行您的 Hadoop 作业(特别是本文中的 Sqoop 作业)，然后一旦您的作业完成，您就可以立即删除集群。这是利用 Dataproc 短暂的按使用付费模型削减成本的一个很好的例子，因为现在您可以快速创建/删除 hadoop 集群，再也不会让集群闲置了！

这里是关于使用 Sqoop 的快速独家新闻(看我在那里做了什么😏)移动数据:

*   Sqoop 将数据从关系数据库系统或大型机导入 HDFS (Hadoop 分布式文件系统)。
*   在 Dataproc Hadoop 集群上运行 Sqoop 可以让您访问内置的云存储连接器，该连接器允许您使用云存储 gs:// file 前缀，而不是 Hadoop hdfs:// file 前缀。
*   前面两点意味着你可以使用 Sqoop 将数据直接导入云存储，完全跳过 HDFS！
*   一旦你的数据在云存储中，你可以简单地使用 Cloud SDK [bq 命令行工具](https://cloud.google.com/bigquery/docs/bq-command-line-tool)将数据[加载到 BigQuery 中。**或者，您可以通过将***Hive . metastore . warehouse . dir***指向一个**](https://cloud.google.com/bigquery/docs/loading-data-cloud-storage-avro#bigquery-import-gcs-file-cli)[**GCS bucket**](https://cloud.google.com/storage/docs/creating-buckets)**，让 Sqoop 将数据直接导入到您的 Dataproc 集群的 Hive 仓库中，该仓库可以基于云存储，而不是 HDFS。**

您可以使用两种不同的方法向集群提交 Dataproc 作业:

## **方法一。)手动 Dataproc 任务提交**

*   [创建 Dataproc 集群](https://cloud.google.com/dataproc/docs/guides/create-cluster)
*   [提交 Dataproc 作业](https://cloud.google.com/dataproc/docs/guides/submit-job)
*   [当作业完成时删除 Dataproc 集群](https://cloud.google.com/dataproc/docs/guides/manage-cluster#deleting_a_cluster)

## **方法二。)使用工作流模板自动提交 Dataproc 任务**

*   创建一个[工作流模板](https://cloud.google.com/dataproc/docs/concepts/workflows/overview)，该模板自动执行之前的 3 个手动步骤(创建、提交、删除)。

使用手动作业提交方法更容易排除作业错误，因为您可以控制何时删除集群。一旦您准备好在生产中运行作业，使用工作流模板的自动化方法是理想的，因为它负责集群创建、作业提交和删除。这两种方法实际上非常类似于设置，下面会有更详细的介绍。

# **等等！在你阅读之前…**

下面的例子演示了使用 Sqoop 连接到 MySQL 数据库。您可以通过在线[quick start for Cloud SQL for MySQL](https://cloud.google.com/sql/docs/mysql/quickstart)在 GCP 快速轻松地创建自己的测试 MySQL 数据库。如果您完成了快速入门，不要忘记使用[云 SQL 代理初始化操作](https://github.com/GoogleCloudPlatform/dataproc-initialization-actions/tree/master/cloud-sql-proxy)创建 Dataproc 集群，以便您可以轻松连接到 MySQL 数据库。最后，通过在完成后清理数据库来避免产生费用。

# 手动提交 Dataproc 作业的 3 个步骤

**1。创建一个 Dataproc 集群**

gcloud 工具的 [Dataproc 集群创建命令](https://cloud.google.com/dataproc/docs/guides/create-cluster)将默认创建一个主节点虚拟机(虚拟机)和两个工作节点虚拟机。所有 3 个节点虚拟机都是 n1 型标准 4 机器。为简单起见，下面的示例命令使用了最少的必要标志。**注意:使用最后一个** `--properties` **标志是为了用云存储位置** `gs://<GCS_BUCKET>/hive-warehouse` **覆盖默认的集群上的 Hive 仓库目录(hdfs:///user/hive/warehouse)。通过将这个 Hive 仓库目录** `hive.metastore.warehouse.dir` **指向云存储，可以持久化所有 Hive 数据(即使删除了 Dataproc 集群)。**

*   *如果您正在连接到本地托管的 MySQL 数据库，因此不需要* [*云 SQL 代理*](https://cloud.google.com/sql/docs/mysql/sql-proxy) *，请使用下面的命令创建您的集群。确保将集群分配给在您的 GCP 环境和本地环境之间共享的* [*VPC 网络*](https://cloud.google.com/vpc/) *。*

```
gcloud dataproc clusters create <**CLUSTER_NAME**> --zone=<**ZONE**> --network=<**VPC_NETWORK**> --properties=hive:hive.metastore.warehouse.dir=gs://<**GCS_BUCKET**>/hive-warehouse
```

*   *如果您使用* [*云 SQL 代理*](https://cloud.google.com/sql/docs/mysql/sql-proxy) *连接到云 SQL-MySQL 数据库，请使用下面的命令创建您的集群。* ***注意:由于默认的 MySQL 端口 3306 已经被 Dataproc 的默认 Hive 元数据 MySQL 服务器占用，所以需要为<端口>选择一个 3306 以外的值(如 3307)。***

```
gcloud dataproc clusters create <**CLUSTER_NAME**> --zone=<**ZONE**> --scopes=default,sql-admin --initialization-actions=gs://dataproc-initialization-actions/cloud-sql-proxy/cloud-sql-proxy.sh --properties=hive:hive.metastore.warehouse.dir=gs://<**GCS_BUCKET**>/hive-warehouse --metadata=enable-cloud-sql-hive-metastore=false --metadata=additional-cloud-sql-instances=<**PROJECT_ID**>:<**REGION**>:<[**SQL_INSTANCE_NAME**](https://cloud.google.com/sql/docs/mysql/instance-info)>=tcp:<**PORT**> 
```

**2。提交一个运行** [**Sqoop 导入工具**](https://sqoop.apache.org/docs/1.4.7/SqoopUserGuide.html#_literal_sqoop_import_literal) 的 dataproc hadoop 作业

*   *如果您希望 Sqoop 将您的数据库表作为 Avro 文件导入云存储，请提交下面的作业命令，然后您将使用 bq 工具* [*加载到 BigQuery*](https://cloud.google.com/bigquery/docs/loading-data-cloud-storage-avro#bigquery-import-gcs-file-cli) *中。*

```
gcloud dataproc jobs submit hadoop --cluster=<**CLUSTER_NAME**> --class=org.apache.sqoop.Sqoop --jars=gs://<**GCS_BUCKET**>/sqoop-1.4.7-hadoop260.jar,gs://<**GCS_BUCKET**>/avro-tools-1.8.2.jar,file:///usr/share/java/mysql-connector-java-5.1.42.jar -- import -Dmapreduce.job.user.classpath.first=true --connect=jdbc:mysql://<**HOSTNAME**>:<**PORT**>/<**DATABASE_NAME**> --username=<**DATABASE_USERNAME**> --password-file=gs://<**GCS_BUCKET**>/passwordFile.txt --target-dir=gs://<**GCS_BUCKET**>/mysql_output --table=<**TABLE**> --as-avrodatafile
```

*   *如果您想让 Sqoop 将您的数据库表导入到一个 Hive 表中，提交下面的 job 命令。* ***注意:下面的命令和上面的一样，只是在*** `— -jars=` ***参数中多加了一个 jar*** `file:///usr/lib/hive/lib/hive-exec.jar` ***，用*** `--hive-import`替换了 `--as-avrodatafile` ***，并去掉了`--target-dir`标志。***

```
gcloud dataproc jobs submit hadoop --cluster=<**CLUSTER_NAME**> --class=org.apache.sqoop.Sqoop --jars=gs://<**GCS_BUCKET**>/sqoop-1.4.7-hadoop260.jar,gs://<**GCS_BUCKET**>/avro-tools-1.8.2.jar,file:///usr/share/java/mysql-connector-java-5.1.42.jar,file:///usr/lib/hive/lib/hive-exec.jar -- import -Dmapreduce.job.user.classpath.first=true --connect=jdbc:mysql://<**HOSTNAME**>:<**PORT**>/<**DATABASE_NAME**> --username=<**DATABASE_USERNAME**> --password-file=gs://<**GCS_BUCKET**>/passwordFile.txt --table=<**TABLE**> --hive-import
```

**3。一旦 Dataproc 任务完成，删除集群**

```
gcloud dataproc clusters delete <**CLUSTER_NAME**>
```

# 仔细看看步骤 2

![](img/99212497900f4f1356d44fd03aa47e8b.png)

近距离观察

`gcloud dataproc jobs submit hadoop`←该命令的第一部分是调用 [gcloud 工具](https://cloud.google.com/sdk/gcloud/reference)的地方。gcloud 工具有很多功能，所以更具体地说，您调用的是 [hadoop](https://cloud.google.com/sdk/gcloud/reference/dataproc/jobs/submit/hadoop) 作业提交命令。该命令嵌套在 gcloud 工具中的几层命令组下: [dataproc 组](https://cloud.google.com/sdk/gcloud/reference/beta/dataproc)->-[作业组](https://cloud.google.com/sdk/gcloud/reference/dataproc/jobs)->-[提交组](https://cloud.google.com/sdk/gcloud/reference/dataproc/jobs/submit) - > [hadoop 命令](https://cloud.google.com/sdk/gcloud/reference/dataproc/jobs/submit/hadoop)。

`--cluster` ←这个 gcloud 标志指定将作业提交到哪个 dataproc 集群。

`--class` ←这个 gcloud 标志指定了您希望 Dataproc 作业运行的主类(org.apache.sqoop.Sqoop)。在幕后，Dataproc 调用 Hadoop 工具并运行这个主类，就像 [Sqoop 命令行工具](https://github.com/apache/sqoop/blob/11c83f68386add243762929ecf7f6f25a99efbf4/bin/sqoop#L101)一样。**注意:确保在** `--jars` **标志中包含主类的 jar 文件的路径(例如** `--jars=gs://<GCS_BUCKET>/sqoop-1.4.7-hadoop260.jar` **)。**

`--jars`←这个 gcloud 标志指定了一个逗号分隔的 jar 文件列表**路径**(路径应该指向 GCS 或 Dataproc 集群中的文件)。这些 jar 文件被提供给 Dataproc 集群中的 MapReduce 和 driver 类路径。要在 Dataproc 中运行 Sqoop，您需要提供以下 jar 文件:

*   最新的 Sqoop 版本([Sqoop-1 . 4 . 7-Hadoop 260 . jar](http://central.maven.org/maven2/org/apache/sqoop/sqoop/1.4.7/sqoop-1.4.7-hadoop260.jar))([Maven 站点](https://mvnrepository.com/artifact/org.apache.sqoop/sqoop)查看最新版本)
*   JDBC 驱动程序需要连接到您的数据库。对于 MySQL 驱动程序，只需将文件路径添加到存在于 Dataproc 集群中的 MySQL 驱动程序(file:///usr/share/Java/MySQL-connector-Java-5 . 1 . 42 . jar)。如果您连接到另一个数据库，例如 DB2，那么包括该数据库所必需的 DB2 JDBC 驱动程序。
*   以指定的输出格式序列化数据所需的任何 jar 文件。在本例中，您将导入的 MySQL 数据存储为 Avro 文件，因此您包括了 Avro jar([Avro-tools-1 . 8 . 2 . jar](https://repo1.maven.org/maven2/org/apache/avro/avro-tools/1.8.2/avro-tools-1.8.2.jar))([Maven 站点](https://mvnrepository.com/artifact/org.apache.avro/avro-tools)以检查最新版本)。**注意:Sqoop 工具本身要求 Avro 版本至少为 1.8.0 才能正常运行，因此即使您不打算以 Avro 格式存储数据，Avro jar 也是必要的。**
*   (可选)已经存在于 Dataproc 中的配置单元驱动程序(file:///usr/lib/Hive/lib/Hive-exec . jar)。**注意:只有当您使用 Sqoop 将数据直接导入 Dataproc 的 Hive 仓库时，才需要这个 Hive jar。**

`-- import`←单个`--` gcloud 参数将左边的 gcloud 特定标志和右边的 hadoop 作业参数分开。既然您将 Sqoop 主类传递给了 hadoop 工具(通过`--class`标志)，那么您还必须指定您希望主类运行的 Sqoop 工具(在本例中为`import`)。在`import`之后，您为 [Sqoop 导入工具](https://sqoop.apache.org/docs/1.4.7/SqoopUserGuide.html#_literal_sqoop_import_literal)指定所有参数。

`-Dmapreduce.job.user.classpath.first=true`←这个 Hadoop 参数对于指示 Hadoop 优先使用用户提供的 jar(通过`--jars`标志传递)而不是集群中包含的默认 jar 是必要的。Sqoop 需要 Avro 的 1.8.0 版本，而 Avro 的本地 Dataproc 版本在撰写本文时是 1.7.7。将这个参数设置为`true`将会给 Sqoop 你在`--jars`列表中传递的 Avro jar 的正确版本。

`-—connect=`←这个 Sqoop 参数指定了 JDBC 连接字符串: *jdbc:mysql:// <主机名> : <端口> / <数据库名>。* **注意:如果您使用云 SQL 代理连接到您的数据库，请确保主机名设置为“localhost ”,并且您使用的端口与您创建 Dataproc 集群**时使用的代理端口相同(例如，确保集群创建参数`--metadata=additional-cloud-sql-instances=<PROJECT_ID>:<REGION>:<[SQL_INSTANCE_NAME](https://cloud.google.com/sql/docs/mysql/instance-info)>=tcp:<**PORT**>`和您的 Sqoop 连接参数`--connect=jdbc:mysql://<HOSTNAME>:<**PORT**>/<DATABASE_NAME>`共享同一个端口)。

`--username=`←这个 Sqoop 参数指定在向关系数据库进行身份验证时使用的用户名。

`--password-file=`←这个 Sqoop 参数指定了 GCS (Google Cloud Storage)中的一个文件的路径，该文件包含您的数据库的密码。**警告:不要使用** `--password` **选项而不是** `--password-file` **选项，因为这将使您的密码暴露在日志中。**

`--target-dir=`←传统上，当在 Hadoop 中运行 Sqoop 时，您会将这个 Sqoop 参数设置为 HDFS (hdfs://)目录路径，但是由于您使用的是 Dataproc，您可以利用内置的 GCS 连接器，将其设置为 GCS (gs://)目录路径。这意味着 Sqoop 会将导入的数据存储在您指定的 GCS 位置，完全跳过 HDFS。

`--table=`←这个 Sqoop 参数指定了要导入的数据库表。

`--as-avrodatafile`←指定此 Sqoop 标志，以 Avro 文件格式存储所有导入的数据。Avro 格式的优点是既能以二进制格式压缩数据，又能将表模式存储在同一个文件中。

`--hive-import` ←指定这个 Sqoop 标志，将所有导入的数据存储到一个 Hive 表中。**注意:当您使用** `--hive-import` **标志时，请确保您的 Dataproc 集群是使用** `--properties=hive:hive.metastore.warehouse.dir=gs://<GCS_BUCKET>/hive-warehouse` **标志创建的，这样您就可以在 GCS 中持久化您的配置单元数据。**

# 使用工作流模板自动提交 Dataproc 任务的 4 个步骤

**1。创建工作流程**

*   *创建工作流模板本身不会创建云 Dataproc 集群，也不会创建和提交作业。模板只是一组指令。只有当工作流模板被* ***实例化*** *时，集群和作业才会被创建和执行(参见步骤 4)。*

```
gcloud beta dataproc workflow-templates create <**TEMPLATE_ID**>
```

**2。添加工作流集群创建属性**

*   *为了简单起见，许多集群属性被故意省略了*

```
gcloud beta dataproc workflow-templates set-managed-cluster <**TEMPLATE_ID**> --zone=<**ZONE**> --cluster-name=<**CLUSTER_NAME**>
```

**3。将作业添加到您的工作流中，以便在集群上运行**

*   *如果您希望 Sqoop 将您的数据库表作为 Avro 文件导入到云存储中，请将下面的作业添加到您的工作流中，然后您将使用 bq 工具* [*加载到 BigQuery*](https://cloud.google.com/bigquery/docs/loading-data-cloud-storage-avro#bigquery-import-gcs-file-cli) *中。*

```
gcloud beta dataproc workflow-templates add-job hadoop --step-id=<**STEP_ID**> --workflow-template=<**TEMPLATE_ID**> --class=org.apache.sqoop.Sqoop --jars=gs://<**GCS_BUCKET**>/sqoop-1.4.7-hadoop260.jar,gs://<**GCS_BUCKET**>/avro-tools-1.8.2.jar,file:///usr/share/java/mysql-connector-java-5.1.42.jar -- import -Dmapreduce.job.user.classpath.first=true --connect=jdbc:mysql://<**HOSTNAME**>:<**PORT**>/<**DATABASE_NAME**> --username=sqoop --password-file=gs://<**GCS_BUCKET**>/passwordFile.txt --target-dir=gs://<**GCS_BUCKET**>/mysql_output --table=<**TABLE**> --as-avrodatafile
```

*   *如果您希望 Sqoop 将您的数据库表导入到配置单元表中，请将下面的作业添加到您的工作流中。* ***注意:下面的命令在*** `— -jars=` ***参数中增加了一个 jar*** `file:///usr/lib/hive/lib/hive-exec.jar` ***并把*** `--as-avrodatafile` ***替换为*** `--hive-import`

```
gcloud beta dataproc workflow-templates add-job hadoop --step-id=<**STEP_ID**> --workflow-template=<**TEMPLATE_ID**> --class=org.apache.sqoop.Sqoop --jars=gs://<**GCS_BUCKET**>/sqoop-1.4.7-hadoop260.jar,gs://<**GCS_BUCKET**>/avro-tools-1.8.2.jar,file:///usr/share/java/mysql-connector-java-5.1.42.jar,file:///usr/lib/hive/lib/hive-exec.jar -- import -Dmapreduce.job.user.classpath.first=true --connect=jdbc:mysql://<**HOSTNAME**>:<**PORT**>/<**DATABASE_NAME**> --username=<**DATABASE_USERNAME**> --password-file=gs://<**GCS_BUCKET**>/passwordFile.txt --table=<**TABLE**> --hive-import
```

**4。实例化/运行工作流**

*   *支持模板的多个(同时)实例化。*

```
gcloud beta dataproc workflow-templates instantiate <**TEMPLATE_ID**>
```

# 将您的 Sqoop-Import 数据加载到 BigQuery 中

![](img/fa5c033c1ace48d3ee39de60e96a93c3.png)

如果您使用 Sqoop 将数据库表导入云存储，您可以使用 [bq 命令行工具](https://cloud.google.com/bigquery/docs/bq-command-line-tool)简单地[将其加载到 BigQuery](https://cloud.google.com/bigquery/docs/loading-data-cloud-storage-avro#bigquery-import-gcs-file-cli) :

```
bq load --source_format=AVRO <**YOUR_DATASET**>.<**YOUR_TABLE**> gs://<**GCS_BUCKET**>/mysql_output/*.avro
```

# 使用 Dataproc 配置单元作业查询配置单元表

![](img/527512cbec5210217f66bb2d0f190a3f.png)

如果您使用 Sqoop 将您的数据库表导入到 Dataproc 中的 Hive，那么您可以通过向 Dataproc 集群提交一个 [Hive 作业](https://cloud.google.com/sdk/gcloud/reference/dataproc/jobs/submit/hive),在您的 Hive 仓库上运行 SQL [查询](https://cwiki.apache.org/confluence/display/Hive/GettingStarted#GettingStarted-SQLOperations):

```
gcloud dataproc jobs submit hive --cluster=<**CLUSTER_NAME**> -e="SELECT * FROM <**TABLE**>"
```

**注意:确保在 Dataproc 集群上运行这些配置单元作业，该集群的默认配置单元仓库目录指向包含您的 Sqoop 导出数据的同一个 GCS bucket(例如，您的集群应该使用** `--properties=hive:hive.metastore.warehouse.dir=gs://<**GCS_BUCKET**>/hive-warehouse` **创建)。**

## 希望这份指南能为你提供许多有价值的数据！