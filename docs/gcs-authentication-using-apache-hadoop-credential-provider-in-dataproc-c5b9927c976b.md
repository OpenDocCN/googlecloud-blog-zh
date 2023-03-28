# 在 Dataproc 中使用 Apache Hadoop 凭证提供程序进行 GCS 认证

> 原文：<https://medium.com/google-cloud/gcs-authentication-using-apache-hadoop-credential-provider-in-dataproc-c5b9927c976b?source=collection_archive---------0----------------------->

这篇文章提供了使用 Apache Hadoop 凭证提供者和 Dataproc 中的服务帐户凭证进行 GCS 认证的概述和示例。对于 Hadoop 用户的应用程序，这种技术可以无缝检索凭证文件中的秘密，这些凭证文件需要访问云存储中的数据集。

# 概观

[GCS 连接器](https://github.com/GoogleCloudDataproc/hadoop-connectors)为 Dataproc Hadoop 应用程序(即 Spark、MapReduce、Hive)来访问存储在外部 GCS 桶中的数据集。默认情况下，对 GCS 的认证和访问不是由提交作业的 Hadoop 用户完成的，而是通过运行 Dataproc 集群的 VM 实例的[主机服务帐户完成的。](https://cloud.google.com/dataproc/docs/concepts/configuring-clusters/service-accounts#service_accounts_in_cloud_dataproc)

[Apache Hadoop 凭证提供者](https://hadoop.apache.org/docs/current/hadoop-project-dist/hadoop-common/CredentialProviderAPI.html)使该机制能够存储和检索加密文件中的秘密(即 JCEKS 文件)放在本地或 Hadoop 文件系统上。此外，GCS 连接器支持 Hadoop 应用程序在访问 GCS 时使用安全存储的凭证进行检索和验证。

![](img/3cb1bc279c0f275a593ed74d5153340d.png)

[图表 1 —高层流程]

上图提供了两个流程，通过 1)集群的默认主机服务帐户和 2)使用存储在 Hadoop 凭据提供程序中的服务帐户机密来区分 Spark 应用程序请求的 GCS 数据访问。在本例中，默认服务帐户`sa-dataproc-instance`没有被授予访问`data-bucket-a`的 IAM 角色，但是`sa-data-access-a`可以访问执行作业所需的数据。提交作业和访问 GCS 中存储的数据集时，序列 1 的流程是默认流程。在序列 2 中，spark 应用程序定义了*Hadoop . security . credential . provider . path*，当应用程序请求从云存储桶中读取数据时，GCS 连接器使用凭据提供程序检索服务帐户凭据，并使用这些凭据进行身份验证以访问`data-bucket-a`。

在这篇博客的后面，我们将详细介绍创建`sa-data-access`服务帐户的步骤，将凭证存储在加密的 JCEKS 文件中，并使用它来访问 Google 云存储。

# 集群安全设计

实施安全措施来保护提供 GCS 数据访问的凭据非常重要。在本节中，我们将回顾在 HDFS 锁定这些凭据的高级原则。

Java 密钥库文件存储使用 Hadoop 凭据实用程序创建的加密服务帐户凭据。该实用程序可以直接创建加密的秘密，并使用默认权限 600 将它们存储在 HDFS 中。应该在文件和父目录上正确设置 HDFS ACL，以便只允许应该访问此凭据的 Hadoop 用户。

如果一个 Dataproc 集群仅由一个租户使用，标准的 GCP 周边安全措施可以在 GCP 项目级和 Dataproc 集群实施，以防止对集群和机密的非法访问。这包括只允许被授权的用户访问项目和 Dataproc 集群，这些用户应该能够访问凭证。此外，确保 Dataproc 作业 API 和 Hadoop APIs，如 HDFS 和 YARN 端点，包括 WebHDFS、HTTP Rest API 等，同样只能由有权访问底层凭证的用户访问。

如果一个 Dataproc 集群有多个租户，那么操作员需要确保启用 OSLogin 来管理 SSH 访问和 sudo 限制，但是还需要通过 Kerberos 启用 Hadoop 安全模式来强制验证 Hadoop 用户。经过身份验证的 Hadoop 用户确保只有被允许对加密的 JCEKS 文件应用 HDFS ACL 的用户才可以访问。如果没有 Kerberos 身份验证，Hadoop 用户可以很容易地模拟其他用户并绕过访问控制。

使用 HDFS 权限和所有权来保护 JCEKS 文件，可以确保只有经过授权的用户才能在访问 GCS 时使用这些凭证。其他授权方法，如 HDFS ACL 和 Apache Ranger 也可用于保护措施。

> *可以通过在 hdfs-site.xml 中将 dfs.permissions.enabled 设置为 true 来启用 HDFS 目录和文件权限/所有权
> 
> * HDFS ACL 扩展了权限/所有权模型，能够为其他用户和组应用访问相同文件/目录的附加权限。可以通过在 hdfs-site.xml 中将 dfs.namenode.acls.enabled 设置为 true 来启用 ACL。

最后，作为额外的最佳实践，可以定期轮换服务帐户密钥。要做到这一点，只需为同一个服务帐户创建一个新的密钥，使用更新后的凭证生成并部署一个新的 JCEKS 文件，并在所有 Hadoop 应用程序切换到使用最新的凭证文件后删除旧的服务帐户密钥。

# 创建和使用凭据

本节将介绍以下步骤:

1.  创建服务帐户`sa-data-access-a`
    *授权通过 IAM 访问 GCS `data-bucket-a`
2.  使用 Hadoop Credential Utility
    * Secure JCEKS file 在 HDFS 创建带有服务帐户凭证的 JCE ks 加密文件
3.  使用 JCEKS 凭据通过 Hadoop fs 命令、MapReduce、Spark 或 Hive 应用程序访问 GC。

# 1.创建服务帐户并授权访问

在第一步中，我们创建一个服务帐户，并为其应用一个 IAM 角色来访问 GCS bucket `data-bucket-a`。

## 创建服务帐户和 JSON 密钥

```
PROJECT_ID=jh-data-sandbox
SERVICE_ACCOUNT=sa-data-access-agcloud iam service-accounts create ${SERVICE_ACCOUNT} \
  --description="sa-data-access-a description" \
  --display-name=${SERVICE_ACCOUNT}gcloud iam service-accounts keys \
  create ~/${SERVICE_ACCOUNT}-key.json \
  --iam-account ${SERVICE_ACCOUNT}@${PROJECT_ID}.iam.gserviceaccount.com
```

## 通过应用 IAM 权限来授权访问

授权服务帐户 *storage.buckets.get* 对 Dataproc 集群的 GCS bucket 的权限:

```
BUCKET=`gcloud dataproc clusters describe dataproc-015 --format="value(config.configBucket)"`gsutil iam ch serviceAccount:${SERVICE_ACCOUNT}@${PROJECT_ID}.iam.gserviceaccount.com:roles/storage.legacyBucketReader gs://${BUCKET}
```

将服务帐户权限授权给数据访问所需的其他 GCS 存储桶:

```
gsutil iam ch serviceAccount:${SERVICE_ACCOUNT}@${PROJECT_ID}.iam.gserviceaccount.com:roles/storage.legacyBucketWriter gs://data-bucket-a
```

# 2.创建 Hadoop 凭据 JCEKS 文件

```
hadoop credential create fs.gs.auth.service.account.email  \
  -provider jceks://hdfs/app/client-app-a/sa-data-access-a.jceks \
  -value "sa-data-access-a@jh-data-sandbox.iam.gserviceaccount.com"hadoop credential create fs.gs.auth.service.account.private.key.id \
  -provider jceks://hdfs/app/client-app-a/sa-data-access-a.jceks \
  -value "0a4e5cb521e2b7c75d082d7069f1cff75071f814"hadoop credential create fs.gs.auth.service.account.private.key \
  -provider jceks://hdfs/app/client-app-a/sa-data-access-a.jceks \
  -value "-----BEGIN PRIVATE KEY-----\n redacted \n-----END PRIVATE KEY-----\n"# apply HDFS ownership to appropriate users and groups requiring access
hadoop fs -chown -R ${USER}:${USER} /app/client-app-a
hadoop fs -chmod 500 /app/client-app-a
hadoop fs -chmod 400 /app/client-app-a/sa-data-access-a.jceks# apply HDFS ACLs (extended acls for advanced setup) for read permission for users and groups requiring access
```

# 3.通过 GCS 连接器使用来自凭据提供者的机密

## Hadoop 客户端

```
hadoop fs -Dhadoop.security.credential.provider.path=jceks://hdfs/app/client-app-a/sa-data-access-a.jceks -ls gs://data-bucket-a/
```

## MapReduce 作业

```
hadoop jar /usr/lib/hadoop-mapreduce/hadoop-mapreduce-examples.jar teragen -Dhadoop.security.credential.provider.path=jceks://hdfs/app/client-app-a/sa-data-access-a.jceks 100000 gs://data-bucket-a/write-data-test/
```

## 火花

```
spark-submit --class org.apache.spark.examples.DFSReadWriteTest \
  --master yarn \
  --deploy-mode cluster \
  --num-executors 3 \
  --conf spark.hadoop.hadoop.security.credential.provider.path=jceks://hdfs/app/client-app-a/sa-data-access-a.jceks \
  /usr/lib/spark/examples/jars/spark-examples.jar \
  /var/log/google-dataproc-agent.0.log \
  gs://data-bucket-a/test-dfs-read-write/
```

## 储备

```
# hive - table x is an external table stored on gs://data-bucket-a/x
hive --hiveconf hadoop.security.credential.provider.path=jceks://hdfs/app/client-app-a/sa-data-access-a.jceks -e 'select count(1) from x;'# beeline 
beeline -u "jdbc:hive2://dataproc-015-m:10000/default;principal=hive/dataproc-015-m@DATAPROC-015.ACME.COM"  --hiveconf hadoop.security.credential.provider.path=jceks://hdfs/app/client-app-a/sa-data-access-a.jceks -e 'select count(1) from x;'
```

# 摘要

总之，我们提供了一个示例来演示在执行 Hadoop 应用程序时，如何使用 Apache Hadoop 凭证提供程序无缝地访问存储在 Java KeyStore 文件中的加密服务帐户凭证。通过使用权限保护 HDFS 的 JCEKS 文件，并通过 Kerberos 执行身份验证，只有被允许使用这些秘密的应用程序和用户才能使用它们来访问云存储中的数据集。这种技术提供了一种机制和额外的选项，可以使用不同的凭证来验证和访问存储在 GCS 中的数据。

# 资源

*   [GCS 连接器 git 库](https://github.com/GoogleCloudDataproc/hadoop-connectors)
*   [GCS 连接器认证配置详情](https://github.com/GoogleCloudDataproc/hadoop-connectors/blob/master/gcs/CONFIGURATION.md#authentication)
*   [data proc 文档中的服务帐户](https://cloud.google.com/dataproc/docs/concepts/configuring-clusters/service-accounts)
*   [Apache Hadoop 凭证提供者指南](https://hadoop.apache.org/docs/current/hadoop-project-dist/hadoop-common/CredentialProviderAPI.html)