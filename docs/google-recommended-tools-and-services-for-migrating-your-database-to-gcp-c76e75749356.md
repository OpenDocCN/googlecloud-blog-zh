# Google 推荐了将数据库迁移到 GCP 的工具和服务

> 原文：<https://medium.com/google-cloud/google-recommended-tools-and-services-for-migrating-your-database-to-gcp-c76e75749356?source=collection_archive---------3----------------------->

*本博客是数据库和存储迁移系列的一部分，旨在帮助您在 Google Cloud 上实现迁移和现代化*

对于任何迁移，一旦选择了要迁移的源，下一步就是选择目的地。如果你不确定 GCP 数据库服务是否完全符合你的需求，你可以根据你的独特需求，使用谷歌官方文档中非常有用的[决策树来选择你想要的 GCP 数据库服务。](https://cloud.google.com/products/databases)

既然您已经决定了要使用的 GCP 数据库选项，本文将向您介绍 Google 推荐的工具/服务来执行迁移。

以下内部资源将作为本指南的一部分，请跳到合适的部分，查看一些合适的 GCP 目的地、推荐的迁移服务及其相对优势和局限性:

1.  [MySQL / PostgreSQL](#9eda)
2.  [SQL 服务器](#16f3)
3.  [甲骨文](#9a8a)
4.  [来自 VMWare 虚拟机的数据库](#0088)
5.  [MongoDB/Redis/Cassandra/Neo4j/inflow DB](#99a9)

# MySQL / PostgreSQL

## 目的地—云 SQL，AlloyDB for PostgreSQL

*推荐的迁移服务* — [数据库迁移服务](https://cloud.google.com/database-migration) *优点/局限性—*

*   本地 MySQL 和 PostgreSQL 迁移免费提供数据库迁移服务。
*   传输完成后，通过手动(对于连续迁移)或自动(对于一次性迁移)升级，目标数据库将成为主数据库。
*   MySQL“系统”数据库不作为服务器迁移的一部分进行迁移，这意味着不包括关于用户角色的信息。
*   不从 PostgreSQL 的物化视图中迁移数据，只迁移视图模式。

## 目的地— BigQuery、云扳手、云 SQL、AlloyDB

*推荐的迁移产品化解决方案* — [Striim](https://cloud.google.com/find-a-partner/partner/striim?redirect=) 【合作伙伴】*优势/局限性—*

*   Striim 专注于通过基于日志的变更数据捕获从企业数据库获取数据流。
*   Striim 具有内嵌转换功能(例如反规格化)以减少延迟。
*   根据[灵活定价模式](https://www.striim.com/pricing/)产生基于许可和基础设施使用的费用。

# SQL Server

## 目标—云 SQL

*推荐迁移服务*——[数据库迁移服务](https://cloud.google.com/database-migration) *优势/局限*—

*   SQL Server 版本的数据库迁移服务正在私人预览中，并将很快正式推出。
*   传输完成后，通过手动(对于连续迁移)或自动(对于一次性迁移)升级，目标数据库将成为主数据库。
*   数据库迁移服务仅迁移带有主键的表。源 SQL Server 数据库中任何没有主键约束的表都不会被迁移。
*   信息模式(information_schema)不作为数据库迁移的一部分进行迁移。

## 目的地— BigQuery、云扳手、云 SQL

*推荐的迁移解决方案—* [Striim](https://cloud.google.com/find-a-partner/partner/striim?redirect=) 【合作伙伴】
*优点/局限性—*

*   Striim 专注于通过基于日志的变更数据捕获从企业数据库获取数据流。
*   Striim 具有内嵌转换功能(例如反规格化)以减少延迟。
*   根据[灵活定价模式](https://www.striim.com/pricing/)产生基于许可和基础设施使用的费用。

## 目标—计算引擎上的云 SQL 实例或 SQL Server

*推荐的迁移方法* — [导入 Microsoft SQL Server 备份](https://cloud.google.com/solutions/migrating-data-between-sql-server-2008-and-cloud-sql-for-sql-server-using-backup-files)—优点/局限性—

*   由于数据库迁移服务仍处于私有预览阶段，并且 Striim 是一个收费的合作伙伴工具，因此考虑将此选项用于中小型工作负载的提升和转移。
*   不支持连续或增量数据迁移。

# 神谕

## 目的地— Google BigQuery、Cloud Spanner、Google Cloud SQL for SQL、MySQL 和 PostgreSQL

*推荐的迁移产品化解决方案* — [Striim](https://cloud.google.com/find-a-partner/partner/striim?redirect=) 【合作伙伴】*优势/局限性—*

*   Striim 专注于通过基于日志的变更数据捕获从企业数据库获取数据流。
*   Striim 具有内嵌转换功能(例如反规格化)以减少延迟。
*   根据[灵活定价模式](https://www.striim.com/pricing/)产生基于许可和基础设施使用的费用。

## 目的地— [裸机解决方案](https://cloud.google.com/bare-metal)

*推荐迁移工具*—[Waverunner](https://github.com/GoogleCloudPlatform/database-migration-wave-orchestrator) 描述—

*   Waverunner 是一个标准化的 Oracle 数据库迁移框架，由 Google 使用[裸机解决方案工具包](https://github.com/google/bms-toolkit)构建。
*   这是一个高度可扩展的框架，有助于以自动化、快速和简单的方式提升和转移 Oracle 工作负载。
*   [裸机解决方案工具包](https://github.com/google/bms-toolkit)获得了 Apache 2 许可。

## 目的地—[PostgreSQL 的云 SQL](https://cloud.google.com/sql)

*推荐迁移服务—* [数据库迁移服务预览](https://cloud.google.com/blog/products/databases/migrate-oracle-to-postgresql) *优点/局限性—*

*   Oracle to Cloud SQL for PostgreSQL 版本的数据库迁移服务正在私人预览中，将很快正式推出。
*   传输完成后，通过手动(对于连续迁移)或自动(对于一次性迁移)升级，目标数据库将成为主数据库。
*   不支持多租户架构(CDB/PDB)、索引组织表(IOT)、临时表等 Oracle 组件。
*   拥有超过 5 亿行的表不能被回填，除非满足某些条件。

## 目的地— [云扳手](https://cloud.google.com/spanner)

*推荐迁移方法* — [迁移指南](https://cloud.google.com/spanner/docs/migrating-oracle-to-cloud-spanner) *注—*

*   Spanner 在概念上不同于其他企业数据库，因此您可能需要重新设计您的应用程序架构，以适应 Spanner 的特性集。
*   Spanner 不支持在数据库级运行用户代码，因此作为迁移的一部分，您必须将由数据库级存储过程和触发器实现的业务逻辑移动到应用程序中。
*   上述指南有助于涵盖从数据模型转换、sql 查询等到从 Oracle 迁移的整个迁移过程。

# 来自 VMWare 虚拟机的数据库

## 目的地——谷歌云计算引擎

*推荐的迁移工具* — [M4CE](https://cloud.google.com/migrate/virtual-machines)

*描述* —如果您有专用虚拟机来托管您的数据库服务器，请选择此选项，您可以将服务器原样迁移到 Google 云计算引擎，以获得更好的可扩展性、安全性和性能。

# 合作伙伴 GCP 数据库服务

## MongoDB 到 MongoDB 图集

*迁移服务*——[MongoDB 实时迁移服务](https://www.mongodb.com/docs/atlas/migration-live-atlas-managed/)

## Redis to Memorystore/Redis 企业云

*迁移服务—* [RDB 备份](https://cloud.google.com/memorystore/docs/redis/import-data) / [Redis 企业解决方案](https://redis.com/redis-enterprise-cloud/migrate/)

## 卡珊德拉致 Datastax Astra

*迁移服务* — [数据税务零停机迁移(ZDM)产品](https://docs.datastax.com/en/astra-serverless/docs/migrate/introduction.html)

## Neo4j 到 Neo4j 光环

*迁移服务—* [Neo4j 管理](https://neo4j.com/docs/operations-manual/3.5/tools/dump-load/)

## InfluxDB 到 InfluxDB 云

移民服务 — [涌入 CLI](https://docs.influxdata.com/influxdb/cloud/migrate-data/migrate-oss/)

我希望这个指南能帮助你了解一些新的服务和建议，把你的数据库迁移到 GCP，如果是的话，请鼓掌:)

敬请关注有关存储迁移的更多信息！