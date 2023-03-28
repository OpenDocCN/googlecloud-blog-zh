# 在 GKE 消费谷歌秘密经理的秘密

> 原文：<https://medium.com/google-cloud/consuming-google-secret-manager-secrets-in-gke-911523207a79?source=collection_archive---------0----------------------->

![](img/0aa79267e647aae98080dfe9dfc5d3d1.png)

在 GKE 消费谷歌秘密经理的秘密

Google Secret Manager(GSM)是 GCP 存储、轮换和检索秘密的旗舰服务。GSM 中的秘密可以是密码、令牌、密钥或您的应用程序需要向端点、数据库或 API 认证的任何随机字符串。

Google Secret Manager 是一个支持 IAM 认证和细粒度访问控制的全局 API。它还提供审计日志(谁…