# 在 Google Cloud Dataproc 集群上为 Hadoop (TDCH)启用 Teradata Connector

> 原文：<https://medium.com/google-cloud/enabling-teradata-connector-for-hadoop-tdch-on-google-cloud-dataproc-cluster-8278017237ec?source=collection_archive---------1----------------------->

曼西·马哈拉纳，迪内什·希利皮蒂雅格(Teradata)

![](img/89b4a4b085c5a3695ec44d8654a96f42.png)

谷歌云 [Dataproc](https://cloud.google.com/dataproc) 是一个完全托管的谷歌云服务，让你利用开源工具，如 Apache Hadoop、Apache Spark、Presto 和许多其他工具进行批处理、查询、流和机器学习。一个常见的用例是能够将 Dataproc 集群连接到一个 Teradata 数据库，并根据用例促进少量到大量数据的移动。在 [Teradata QueryGrid](https://www.teradata.com/Products/Ecosystem-Management/IntelliSphere/QueryGrid) 数据连接功能套件下，Teradata 为 Dataproc 提供了多种连接选项，包括[Teradata Connector for Hadoop(TDCH)](https://downloads.teradata.com/download/connectivity/teradata-connector-for-hadoop-tdch-command-line-edition)，这将是本文的重点。

[TDCH](https://downloads.teradata.com/download/connectivity/teradata-connector-for-hadoop-tdch-command-line-edition) 是一款基于 Hadoop 的数据集成工具，支持 Teradata vantage 系统和各种 Hadoop 生态系统组件之间的高性能并行双向数据移动和转换。它为客户提供了一套丰富的可扩展内置处理能力，以便在数据量达到峰值时执行复杂的数据集成流程。

在这篇博文中，我们将学习如何在 Dataproc 上安装、配置和使用 TDCH 软件，同时强调兼容性考虑和最佳实践。

## 版本

作为先决条件，了解 TDCH 和 Dataproc 的支持版本是很重要的。为了支持多种 Hadoop 版本，TDCH 有 5 个活跃的分支— 1.5.x、1.6.x、1.7.x、1.8.x 和 1.9.x

其中只有两个分支适用于 Dataproc，如下所示。

TDCH 1.5.12 目前已经过测试和认证，可用于:

*   Google Cloud data proc 1.4 . x(Debian 10，Hadoop 2.9)
*   谷歌云 Dataproc 1.4.x (Ubuntu 18.04 LTS，Hadoop 2.9)
*   Google Cloud data proc 1.5 . x(CentOS 8，Hadoop 2.10)
*   Google Cloud data proc 1.5 . x(Debian 10，Hadoop 2.10)
*   谷歌云 Dataproc 1.5.x (Ubuntu 18.04 LTS，Hadoop 2.10)

TDCH 1.9.0 目前已经过测试和认证，可用于:

*   Google Cloud data proc 2.0 . x(CentOS 8，Hadoop 3.2.2)
*   谷歌云 Dataproc 2.0.x (Debian 10，Hadoop 3.2.2)
*   谷歌云 Dataproc 2.0.x (Ubuntu 18.04 LTS，Hadoop 3.2.2)

确定兼容性的最佳方式是阅读您想要安装的版本的自述文件中的“Google Cloud Dataproc 建议”部分，此处提供[此处提供](https://downloads.teradata.com/download/connectivity/teradata-connector-for-hadoop-command-line-edition)，您也可以参考 SUPPORTLIST 链接了解更多信息。

## 安装说明

注意:TDCH 需要安装在 Dataproc 1.x 集群的所有节点上。这个要求只是 Dataproc 1.x 独有的。

1.  从[这里](https://downloads.teradata.com/download/connectivity/teradata-connector-for-hadoop-command-line-edition)下载你要安装版本的 Teradata connector rpm 文件。

2.对于 Debian/Ubuntu 来说，

>使用以下命令在 Debian/Ubuntu 平台上安装 TDCH rpm 文件:
$ sudo apt-get install alien
$ sudo alien teradata-connector-<version>…。每分钟转数

>复制上面生成的。deb 文件(TERADATA_CONNECTOR_FILE)到 GCS 存储桶位置(DOWNLOAD_LOCATION 的值)

3.对于 CentOS 来说，

>将上述 rpm 文件(TERADATA_CONNECTOR_FILE)复制到一个 GCS 桶位置(DOWNLOAD_LOCATION 的值)

4.将初始化动作脚本从[复制到 GCS 位置。这只是一个示例，修改它以满足您的需要。](https://github.com/mansim07/initialization-actions/blob/master/tdch/tdch.sh)

5.使用以下命令创建 Dataproc 集群:

对于 Dataproc 1.x

```
REGION=<region>
CLUSTER=<cluster_name>gcloud dataproc clusters create${CLUSTER} **\***--*region=${REGION}**\**--initialization-actions=gs://***<bucket-name-with-path>***/tdch.sh**\**--enable-component-gateway **\**
*--*metadata=TERADATA_CONNECTOR_FILE="<***name-of-deb-or-rpm-file****>"***\** --metadata=DOWNLOAD_LOCATION="<***gcs-bucket-path-download-location>"* \** --metadata=LIB_TO_BE_ADDED="file:/usr/lib/tdch/*<****version****>*/lib***"* \**--metadata=TEZ_XML_FILE="/etc/tez/conf/tez-site.xml***"***
```

对于 Dataproc 2.x 和更高版本

```
REGION=<region>
CLUSTER=<cluster_name>gcloud dataproc clusters create${CLUSTER} **\***--*region=${REGION}**\**--initialization-actions=gs://***<bucket-name-with-path>***/tdch.sh**\**--enable-component-gateway **\**
*--*metadata=TERADATA_CONNECTOR_FILE="<***name-of-deb-or-rpm-file****>"***\** --metadata=DOWNLOAD_LOCATION="<***gcs-bucket-path-download-location>"***
```

要了解如何在 Dataproc 中使用初始化动作，请点击这里的。

以上步骤主要集中在使用初始化操作创建 Dataproc 集群期间的 TDCH 安装上。创建群集后，您还可以运行步骤 1、2 或 3(取决于操作系统版本)来手动安装 TDCH。

**注:**

*   使用 alien 将 rpm 转换为 deb 的第 2 步可以在外部完成(从远程主机)。主机可以是 Windows (WSL)/Mac/Linux 等。
*   alien 软件包可能并非在所有平台上都可用，因此在选择合适的外部主机平台时请谨慎。
*   如果希望转换更加自动化，而不是将 rpm 转换为 deb 并将 deb 文件上传到 GCS bucket，您可以直接将 rpm 上传到 GCS bucket，并通过修改脚本来满足您的需求，让初始化脚本负责基于 OS 的 rpm 到 deb 的转换。

## **在 Dataproc 上执行 TDCH 作业**

1.  第一步是在作业提交者节点上设置环境变量。依赖的 jar 版本应该与 Dataproc 集群上可用的版本一致。

> 注意:**【文件://】***前缀仅在从远程主机提交作业时需要。*

```
export TDCH_JAR=/usr/lib/tdch/<***version***>/lib/teradata-connector-<***version***>.jarexport HADOOP_HOME=***[file://]***/usr/lib/hadoopexport HIVE_HOME=***[file://]***/usr/lib/hiveexport HCAT_HOME=***[file://]***/usr/lib/hive-hcatalogexport HADOOP_CLASSPATH=$HIVE_HOME/conf:/etc/tez/conf:$HIVE_HOME/lib/antlr-runtime-<***version-on-cluster***>.jar:$HIVE_HOME/lib/commons-dbcp-<***version-on-cluster***>.jar:$HIVE_HOME/lib/commons-pool-<***version-on-cluster***>.jar:$HIVE_HOME/lib/datanucleus-core-<***version-on-cluster***>.jar:$HIVE_HOME/lib/datanucleus-rdbms-<***version-on-cluster***>.jar:$HIVE_HOME/lib/hive-cli-<***version-on-cluster***>.jar:$HIVE_HOME/lib/hive-exec-<***version-on-cluster***>.jar:$HIVE_HOME/lib/hive-metastore-<***version-on-cluster***>.jar:$HIVE_HOME/lib/jdo-api-<***version-on-cluster***>.jar:$HIVE_HOME/lib/libfb303-<***version-on-cluster***>.jar:$HIVE_HOME/lib/libthrift-<***version-on-cluster***>.jar:$HIVE_HOME/lib/lz4-<***version-on-cluster***>.jar:$HCAT_HOME/share/hcatalog/hive-hcatalog-core-<***version-on-cluster***>.jar:$HADOOP_HOME/lib/hadoop-lzo-<***version-on-cluster***>.jar:***[file://]***/usr/lib/tdch/<***version***>/lib/tdgssconfig.jar:***[file://]***/usr/lib/tdch/<***version***>/lib/terajdbc4.jar:***[file://]***/usr/lib/spark/jars/avro-mapred-<***version-on-cluster***>-hadoop2.jar:***[file://]***/usr/lib/spark/jars/paranamer-<***version-on-cluster***>.jar:***[file://]***/usr/lib/tez/tez-api-<***version-on-cluster***>.jarexport LIB_JARS=$HIVE_HOME/lib/hive-cli-<***version-on-cluster***>.jar,$HIVE_HOME/lib/hive-exec-<***version-on-cluster***>.jar,$HIVE_HOME/lib/hive-metastore-<***version-on-cluster***>.jar,$HIVE_HOME/lib/libfb303-<***version-on-cluster***>.jar,$HIVE_HOME/lib/libthrift-<***version-on-cluster***>.jar,$HIVE_HOME/lib/jdo2-api-<***version-on-cluster***>.jar,***[file://]***/usr/lib/tdch/<***version***>/lib/tdgssconfig.jar,***[file://]***/usr/lib/tdch/<***version***>/lib/terajdbc4.jar
```

请注意，根据您的需求，可能需要其他依赖库。您可以在集群上找到它们，如果它们是运行时依赖项，可以将它们添加到 HADOOP_CLASSPATH 或 LIB_JAR 中。

2.提交 TDCH 作业的示例命令

Teradata Connector for Hadoop 支持以下数据传输方法:

*   连接器导入工具(Teradata 到 HDFS/Hive/HCat)
*   连接器导出工具(HDFS/Hive/HCat 到 Teradata)

要在 Dataproc VM 中运行它，

```
hadoop jar $TDCH_JAR \ com.teradata.connector.common.tool.***ConnectorImportTool*** \
-classname com.teradata.jdbc.TeraDriver \
-url ***jdbc:teradata://sampledbserver/DATABASE=sampledb*** \
-username ***sampleuser*** \
-password ***samplepwd*** \
-jobtype hive \
-fileformat parquet \
-sourcetable ***db.customer*** \
-sourcefieldnames “***id,acctnum,…***” \
-targettable ***db.cust*** \
-targetfieldnames “***id,acctnum,…***”
```

要使用 gcloud 或其他远程节点运行它，

```
gcloud dataproc jobs submit hadoop \
 --cluster=<***cluster-name***> \
 --region <***region***> \
--class=com.teradata.connector.common.tool.***ConnectorImportTool*** \
--jars=$TDCH_JAR,$HADOOP_CLASSPATH \
-- -libjars $LIB_JARS \
-classname com.teradata.jdbc.TeraDriver \
-url ***jdbc:teradata://sampledbserver/DATABASE=sampledb*** \
-username ***sampleuser*** \
-password ***samplepwd*** \
-jobtype hive \
-fileformat parquet \
-sourcetable ***db.customer*** \
-sourcefieldnames “***id,acctnum,..***” \ 
-targettable ***db.cust*** \
-targetfieldnames “***id,acctnum,…***”
```

要了解有关连接器和可用特性和功能的更多信息，请参考此[链接](https://downloads.teradata.com/download/connectivity/teradata-connector-for-hadoop-command-line-edition)上的连接器自述文件以及此[链接](https://support.teradata.com/community?id=community_question&sys_id=98bb47631bd7fb00682ca8233a4bcb2a)上的 TDCH 教程。

## **在谷歌云中运行 TDCH 的注意事项**

此[链接](https://downloads.teradata.com/download/connectivity/teradata-connector-for-hadoop-command-line-edition)提供的自述文件的第 9 部分突出显示了更全面的注意事项列表。在这里，我们将强调一些需要注意的关键事项:

*   专门针对谷歌云，

>虚拟机实例所属的防火墙规则必须为 Teradata 数据库的入站和出站流量开放端口 1025

>如果使用 internal.fastexport 和/或 internal.fastload 协议，它们的默认端口 8678 和 65535 需要为入口和出口开放

*   TDCH 需要安装在 Dataproc 1.x 集群的所有节点上。这一要求只有 Dataproc 1.x 才有。
*   对于生产用途，在创建集群之前，强烈建议您将初始化操作复制到您自己的云存储桶中，以保证在所有 Dataproc 集群节点上一致使用相同的初始化操作代码，并防止集群中上游的意外升级。这里的[是指](https://github.com/GoogleCloudDataproc/initialization-actions)。

## 包裹

虽然上面的大部分步骤可以手动执行，但是使用 Dataproc 初始化操作可以降低维护开销，尤其是对于短暂的 Dataproc 集群。

特别感谢来自 Teradata 的 Mohan Talla 的所有帮助和支持。

希望这个技术指南对你有所帮助！