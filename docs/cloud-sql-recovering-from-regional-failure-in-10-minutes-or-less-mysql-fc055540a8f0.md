# 云 SQL:在 10 分钟或更短时间内从区域故障中恢复(MySQL 和 PostgresSQL)

> 原文：<https://medium.com/google-cloud/cloud-sql-recovering-from-regional-failure-in-10-minutes-or-less-mysql-fc055540a8f0?source=collection_archive---------1----------------------->

> 更新:您现在可以在 2 分钟或更短时间内执行跨区域故障转移！点击这里查看我对本文的[更新，其中详细介绍了如何使用新的云 SQL 特性来实现这一恢复时间！](/google-cloud/cloudsql-cross-region-ha-just-got-easier-and-whole-lot-faster-4f94c7dcc71d)

# 介绍

Google Cloud SQL 原生提供了一个[完全托管的高可用性服务](https://cloud.google.com/sql/docs/mysql/high-availability)，当在一个实例上启用时，可以在单个区域的多个区域中提供数据冗余。这意味着，如果您的主实例停机，云 SQL…