# 在 GCP 配置云函数错误的警报通知

> 原文：<https://medium.com/google-cloud/configuring-alert-notifications-for-cloud-function-errors-in-gcp-c47cd83e2ac5?source=collection_archive---------0----------------------->

在 GCP 运行云功能时，收到关键错误警报是一种有用且常见的需求。如果您的系统是活动的，并且您不需要支持管理员来检查和查看 Stackdriver 错误，这将非常有用。

整个过程分为以下主要步骤，

1.  为您想要监控的特定错误创建一个 Stackdriver“日志度量”过滤
2.  设置通知渠道
3.  创建警报策略