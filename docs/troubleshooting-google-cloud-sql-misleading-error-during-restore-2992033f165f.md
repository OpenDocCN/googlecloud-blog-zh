# 恢复期间 Google Cloud SQL“误导性”错误故障排除

> 原文：<https://medium.com/google-cloud/troubleshooting-google-cloud-sql-misleading-error-during-restore-2992033f165f?source=collection_archive---------0----------------------->

**Google Cloud SQL 误导性错误:(g Cloud . SQL . backups . restore)HTTPError 403:客户端未被授权进行此请求。**

***错误:(g cloud . SQL . backups . restore)HTTPError 403:客户端无权进行此请求。***

![](img/e251819496f05d4f3949cdc71fa7d164.png)

如果您在恢复云 SQL 备份时出现上述错误，这肯定表明相关 IAM 用户缺乏恢复备份所需的权限，但有时此错误消息具有误导性，根本原因可能是其他原因。

初步看来，这似乎是一个权限问题——恢复到不同项目的用户必须拥有目标项目的*cloud SQL . instances . restore backup*权限和源实例的 *cloudsql.backupRuns.get* 权限。这些权限包含在云 SQL 管理员角色中。

博客的范围仅限于同一个项目，因此让我们在现有云 SQL 实例上重新创建场景:

```
***$ gcloud sql instances list******NAME DATABASE_VERSION LOCATION TIER PRIMARY_ADDRESS PRIVATE_ADDRESS STATUS******pg-db11 POSTGRES_10 asia-south1-a db-custom-8–32768 XX.XXX.XX.XXX 10.45.193.230 RUNNABLE******mymart2 MYSQL_5_7 asia-south1-a db-custom-4–26624–10.45.193.3 STOPPED******$***
```

假设我将把 *pg-db11* 的备份恢复到另一个实例 pg-db22，然后让我们列出成功的备份，并选择其中一个备份进行恢复。

```
***$ gcloud sql backups list — instance pg-db11******ID WINDOW_START_TIME ERROR STATUS INSTANCE******1660312099586 2022–08–12T13:48:19.586+00:00 — SUCCESSFUL pg-db11******1659712251075 2022–08–05T15:10:51.075+00:00 — SUCCESSFUL pg-db11***
```

让我们验证实例是否存在:

```
***$ gcloud sql instances list|grep pg-db22******$***
```

以上结果表明目标实例不存在，如果我们尝试恢复该实例会怎样:

```
***$ gcloud sql backups restore 1660312099586 \******> — restore-instance=pg-db22 \******> — backup-instance=pg-db11******All current data on the instance will be lost when the backup is restored.******Do you want to continue (Y/n)? Y******ERROR: (gcloud.sql.backups.restore) HTTPError 403: The client is not authorized to make this request.******$***
```

上述错误表明当前 gcloud 用户缺乏所需的权限。因此，让我们检查用户是否拥有所需的权限:

首先确定当前 gcloud 环境的用户和服务帐户:

```
***$ gcloud auth list******Credentialed Accounts******ACTIVE ACCOUNT******* 664290125703-compute@developer.gserviceaccount.com******admin@shkm.altostrat.com******To set the active account, run:******$ gcloud config set account `ACCOUNT`******$***
```

检查上述用户是否具有云 sql 管理员角色。下面是上述输出的摘录，表明它已经具有所需的权限:

```
***$ gcloud projects get-iam-policy shailesh-1****[output is truncated here for better visibility]*
.
.
.
.
.***role: roles/cloudsql.admin******- members:******- serviceAccount:664290125703-compute@developer.gserviceaccount.com******- user:admin@shkm.altostrat.com******role: roles/cloudsql.editor******- members:******- user:admin@shkm.altostrat.com******role: roles/cloudsql.instanceUser******- members:******- serviceAccount:664290125703-compute@developer.gserviceaccount.com****[output is truncated here for better visibility]*
.
.
.
.
.
.***version: 1******$***
```

上述输出确认不存在权限问题，因为当前用户是" ***角色:roles/cloudsql.admin"*** 的一部分。

让我们看看，根据最佳实践，我们是否已经有了一个目标实例。当您将备份恢复到不同的实例时，请记住以下限制和最佳做法:

1.  目标实例必须与从中获取备份的实例具有相同的数据库版本。
2.  目标实例的存储容量必须至少与正在备份的实例的容量一样大。使用的存储量无关紧要。您可以在控制台云 SQL 实例页面中看到实例的存储容量
3.  目标实例必须处于可运行状态。
4.  目标实例的核心数或内存量可以不同于从中进行备份的实例。
5.  目标实例可以与源实例位于不同的区域。
6.  在停机期间，您仍然可以检索特定项目中的备份列表。

让我们验证目标云 SQL SQL 实例 pg-db22 是否存在:

```
***$ gcloud sql instances list|grep pg-db22******$***
```

上面的输出表明这个实例不存在，所以让我们按照上面指定的最佳实践来创建:目标实例必须具有与从中获取备份的实例相同的数据库版本。

```
***$ gcloud sql instances create pg-db22 — database-version=POSTGRES_10 — cpu=4 — memory=16GiB — zone=asia-south1-a — root-password=Welcome0******Creating Cloud SQL instance…done.******Created [https://sqladmin.googleapis.com/sql/v1beta4/projects/shailesh-1/instances/pg-db22].******NAME DATABASE_VERSION LOCATION TIER PRIMARY_ADDRESS PRIVATE_ADDRESS STATUS******pg-db22 POSTGRES_10 asia-south1-a db-custom-4–16384 XX.XXX.XX.XXX — RUNNABLE******$***
```

让我们验证:

```
***$ gcloud sql instances list|grep pg-db22******pg-db22 POSTGRES_10 asia-south1-a db-custom-4–16384 XX.XXX.XX.XXX — RUNNABLE******$***
```

现在，让我们使用之前出现错误的命令进行恢复:

```
***$ gcloud sql backups restore 1660312099586 — restore-instance=pg-db22 — backup-instance=pg-db11******All current data on the instance will be lost when the backup is restored.******Do you want to continue (Y/n)? Y******Restoring Cloud SQL instance…done.******Restored [https://sqladmin.googleapis.com/sql/v1beta4/projects/shailesh-1/instances/pg-db22].******$***
```

耶，现在成功了:

```
***$ gcloud sql instances list******NAME DATABASE_VERSION LOCATION TIER PRIMARY_ADDRESS PRIVATE_ADDRESS STATUS******pg-db22 POSTGRES_10 asia-south1-a db-custom-4–16384 XX.XXX.XX.XXX — RUNNABLE******pg-db11 POSTGRES_10 asia-south1-a db-custom-8–32768 XX.XXX.XX.XXX 10.45.193.230 RUNNABLE******mymart2 MYSQL_5_7 asia-south1-a db-custom-4–26624–10.45.193.3 STOPPED******$***
```

希望这有所帮助。