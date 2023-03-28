# 如何用 terraform 创建 CloudSQL Mysql 实例

> 原文：<https://medium.com/google-cloud/how-to-create-cloudsql-mysql-instance-with-terraform-e87a396b2b58?source=collection_archive---------0----------------------->

先决条件:

你需要地形

参见[https://learn.hashicorp.com/tutorials/terraform/install-cli](https://learn.hashicorp.com/tutorials/terraform/install-cli)安装平台。

1-创建 tfvar 文件:

```
#cat terraform.tfvarsproject = "project-id"region = "europe-west4"
```

2-创建 tf 文件，并将 mysql 数据库相关信息放入其中，例如实例、数据库名称和 root 密码:

```
#cat main.tfvariable "project" {}variable "region" {}provider "google" {project = "${var.project}"region = "${var.region}"}resource “google_sql_database_instance” “master” {name = "instance_name"database_version = "MYSQL_5_7"region = "${var.region}"settings {tier = "db-n1-standard-2"}}resource "google_sql_database" “database” {name = "database_name"instance = "${google_sql_database_instance.master.name}"charset = "utf8"collation = "utf8_general_ci"}resource "google_sql_user" "users" {name = "root"instance = "${google_sql_database_instance.master.name}"host = "%"password = "XXXXXXXXX"}
```

3-在 gcp 上创建服务帐户，该帐户具有创建 CloudSQL 实例的必要权限:

```
gcloud iam service-accounts create sa-terraform — display-name "Terraform service account"
```

将 CloudSQL Admin 角色分配给服务帐户:

```
gcloud projects add-iam-policy-binding PROJECT_ID — member="serviceAccount:sa-terraform@PROJECT_ID.iam.gserviceaccount.com" — role="roles/cloudsql.admin"
```

创建 terraform 将使用的服务帐户密钥:

```
gcloud iam service-accounts keys create ./key.json \— iam-account sa-terraform[@](mailto:terraform-sa@project-id.iam.gserviceaccount.com)PROJECT_ID[.iam.gserviceaccount.com](mailto:terraform-sa@project-id.iam.gserviceaccount.com)
```

导出 GOOGLE _ 应用程序 _ 凭据:

```
export GOOGLE_APPLICATION_CREDENTIALS=./key.json
```

4-初始化地形:

```
terraform init
```

5-获取地形图

```
terraform plan
```

6-应用地形

```
terraform apply
```

如果这是一个全新的项目，或者如果您之前没有在项目中创建 CloudSQL，您可能会得到如下错误:

## 错误:错误，未能创建实例 emlak: googleapi:错误 403:云 SQL Admin API 以前未在项目 XXXXXXXXX 中使用过，或者已被禁用。通过访问[https://console . developers . Google . com/APIs/API/sqladmin . Google APIs . com/overview 启用？project=XXXXXXXXX](https://console.developers.google.com/apis/api/sqladmin.googleapis.com/overview?project=XXXXXXXXX) 然后重试。如果您最近启用了此 API，请等待几分钟，让操作传播到我们的系统，然后重试。，访问未配置

> 您可以在 GCP 控制台中启用 Cloudsql api，或者只需将错误消息中的 url 复制/粘贴到您的浏览器中，然后启用该 api。

然后再次尝试 terraform 应用。

```
Do you want to perform these actions?
 Terraform will perform the actions described above.
 Only ‘yes’ will be accepted to approve.Enter a value: yesgoogle_sql_database_instance.master: Creating…
google_sql_database_instance.master: Still creating… [10s elapsed]
google_sql_database_instance.master: Still creating… [20s elapsed]
google_sql_database_instance.master: Still creating… [30s elapsed]
google_sql_database_instance.master: Still creating… [40s elapsed]
google_sql_database_instance.master: Still creating… [50s elapsed]
google_sql_database_instance.master: Still creating… [1m0s elapsed]
google_sql_database_instance.master: Still creating… [1m10s elapsed]
google_sql_database_instance.master: Still creating… [1m20s elapsed]
google_sql_database_instance.master: Still creating… [1m30s elapsed]
google_sql_database_instance.master: Still creating… [1m40s elapsed]
google_sql_database_instance.master: Still creating… [1m50s elapsed]
google_sql_database_instance.master: Still creating… [2m0s elapsed]
google_sql_database_instance.master: Still creating… [2m10s elapsed]
google_sql_database_instance.master: Still creating… [2m20s elapsed]
google_sql_database_instance.master: Still creating… [2m30s elapsed]
google_sql_database_instance.master: Still creating… [2m40s elapsed]
google_sql_database_instance.master: Still creating… [2m50s elapsed]
google_sql_database_instance.master: Creation complete after 2m55s [id=instanceid]
google_sql_database.database: Creating…
google_sql_user.users: Creating…
google_sql_user.users: Still creating… [22s elapsed]
google_sql_database.database: Still creating… [22s elapsed]
google_sql_database.database: Creation complete after 24s [id=projects/projectid/instances/instanceid/databases/dbid]
google_sql_user.users: Creation complete after 28s [id=root/%/instanceid]
```

就是这样，你用 terraform 创建了你的 Cloudsql Mysql 实例。