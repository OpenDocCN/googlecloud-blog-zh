# 在 Google Kubernetes 引擎上持续交付 HashiCorp Vault:备份和恢复

> 原文：<https://medium.com/google-cloud/continuous-delivery-of-hashicorp-vault-on-google-kubernetes-engine-backup-recovery-725e5e1f6b48?source=collection_archive---------0----------------------->

**这是一个系列的第 8 部分:** [**指数**](/@blzysh/continuous-delivery-of-hashicorp-vault-on-google-kubernetes-engine-bcbf4e75f0f6)

# 概观

为了备份 GCS 数据，我们启用了[对象版本控制](https://cloud.google.com/storage/docs/object-versioning)，这将允许我们将单个对象恢复到以前的版本。

您可以使用 gsutil 查看版本:

```
gsutil ls -lra gs://ops-bcurtis-sb-vault-storage/core/mounts617  2018-10-11T02:05:26Z  gs://ops-bcurtis-sb-vault-storage/core/mounts#1539223526242045  metageneration=1
690  2018-10-11T02:05:26Z  gs://ops-bcurtis-sb-vault-storage/core/mounts#1539223526479479  metageneration=1
771  2018-10-29T18:50:20Z  gs://ops-bcurtis-sb-vault-vault-storage/core/mounts#1540839020037493  metageneration=1
TOTAL: 3 objects, 2078 bytes (2.03 KiB)
```

您可以看到该文件的三个版本，任何一个都可以使用 copy 恢复:

```
gsutil cp gs://ops-bcurtis-sb-vault-storage/core/mounts#15392235262
42045 gs://ops-bcurtis-sb-vault-storage/core/mounts
Copying gs://ops-bcurtis-sb-vault-storage/core/mounts#1539223526242045 [Content-Type=application/octet-stream]...
/ [1 files][  617.0 B/  617.0 B]
```

接下来，您可以看到现在有四个版本:

```
gsutil ls -lra gs://ops-bcurtis-sb-vault-storage/core/mounts
617  2018-10-11T02:05:26Z  gs://ops-bcurtis-sb-vault-storage/core/mounts#1539223526242045  metageneration=1
690  2018-10-11T02:05:26Z  gs://ops-bcurtis-sb-vault-storage/core/mounts#1539223526479479  metageneration=1
771  2018-10-29T18:50:20Z  gs://ops-bcurtis-sb-vault-storage/core/mounts#1540839020037493  metageneration=1
617  2018-12-04T14:50:46Z  gs://ops-bcurtis-sb-vault-storage/core/mounts#1543935046514223  metageneration=1
TOTAL: 4 objects, 2695 bytes (2.63 KiB)
```

[云存储转移服务](https://cloud.google.com/storage-transfer/docs/overview)用于将云存储桶从源项目备份到目的备份项目 ops-bcurtis-sb-vault-backup。每天早上 8:00 运行。如果您正在进行升级或其他可能对数据存储产生负面影响的操作，您可以禁用传输作业，这样就不会同步坏数据。一旦事情得到验证，再次启用同步

如果我们不小心删除了 KMS 密钥，会有一个小的恢复窗口，我们可以立即联系我们的 TAM/支持代表。

# 恢复

# 数据存储(谷歌云存储)

为了恢复，我们可以在备份云存储桶和目标保管库存储桶之间运行 rsync:

```
gstuil rsync -r gs://SOURCE_BUCKET_NAME gs://DESTINATION_BUCKET_NAME
```

# 项目被破坏

为了从意外破坏或导致谷歌云项目不可用或丢失的事件中恢复，我们需要联系谷歌支持来恢复 KMS 密钥。然后，您需要构建一个新的环境(切换到恢复的密钥)并如上所述同步所有内容。