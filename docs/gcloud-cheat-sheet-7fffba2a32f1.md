# gcloud 备忘单

> 原文：<https://medium.com/google-cloud/gcloud-cheat-sheet-7fffba2a32f1?source=collection_archive---------0----------------------->

没有深刻的见解或故事可讲。只是一个我不断寻找的`gcloud`命令列表，我想我应该把它放在一个公共的、可访问的地方。

这将不断被添加到。记住，这是在你通过`gcloud init`登录的任何项目的上下文中，所以也要检查一下，这样你就不会得到意想不到的结果。

# 一般项目材料

我以谁的身份登录？
`gcloud auth list`

我的项目名称是什么？


我的项目号是多少？
`gcloud projects list --filter="$(gcloud config get-value project)" --format="value(PROJECT_NUMBER)"`

我的项目 ID 是什么？


# 演员表

把我所有的账单都输入 CSV。
`gcloud alpha billing accounts list --format="csv(displayName,masterBillingAccount,name,open)" > billingAccounts.csv`

# InternationalAssociationofMachinists 国际机械师协会

将 secret.mangaer/secretAccessor 添加到我的云构建服务帐户。
`gcloud projects add-iam-policy-binding $(gcloud config get-value project) --member=serviceAccount:"$(gcloud projects list --filter="$(gcloud config get-value project)" --format="value(PROJECT_NUMBER)")@cloudbuild.gserviceaccount.com" --role=roles/secretmanager.secretAccessor`

将`roles/cloudfunctions.invoker`添加到远程连接的 BigQuery 生成的服务帐户中。

# 整洁的东西

为我的基础架构生成 Terraform HCL。
`gcloud alpha resource-config bulk-export --resource-format=terraform > allstuff.tf`

获取名称与子字符串匹配的虚拟机的名称。
`gcloud compute instances list --filter="my_partial_string-" --format="value(NAME)"`