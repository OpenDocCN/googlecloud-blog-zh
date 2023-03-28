# Liquibase 扳手加长件

> 原文：<https://medium.com/google-cloud/liquibase-spanner-extension-6df18d3829f3?source=collection_archive---------4----------------------->

# 介绍

我们非常高兴地宣布 Liquibase 扳手扩展测试版 1.0 的可用性。这为 Spanner 带来了 Liquibase 的所有 CI/CD 优势。

您可以在 GitHub 中找到源代码和详细信息:

[](https://github.com/cloudspannerecosystem/liquibase-spanner) [## cloudspanner 生态系统/liqui base-扳手

### Liquibase 扩展增加了对 Google Cloud Spanner 的支持。将它包含在您的应用程序项目中以运行…

github.com](https://github.com/cloudspannerecosystem/liquibase-spanner) 

# 支持的功能

扩展支持以下更改类型: *createTable、dropTable、addColumn、modifyDataType、addNotNullConstraint、dropColumn、createIndex、dropIndex、addForeignKeyConstraint、dropForeignKeyConstraint、dropAllForeignKeyConstraints、addLookupTable、insert、update、loadData 和 loadUpdateData。*

ChangeTypes 都是作为单元测试来测试的，用的是[扳手模拟器](https://cloud.google.com/spanner/docs/emulator)，用的是真正的扳手。

# 例子

提供了一个示例 [changelog.yaml](https://github.com/cloudspannerecosystem/liquibase-spanner/blob/master/example/changelog.yaml) ，演示了一个应用于 Spanner 的 changelog。这有助于使用扳手应用 [Liquibase 最佳实践](https://www.liquibase.org/get-started/best-practices)。

它包括以下内容:

*   [create-schema.yaml](https://github.com/cloudspannerecosystem/liquibase-spanner/blob/master/example/create-schema.yaml) :为歌手和专辑创建表格，包括交叉表、列选项和索引。
*   [load-data-singers.yaml](https://github.com/cloudspannerecosystem/liquibase-spanner/blob/master/example/load-data-singers.yaml) :将数据从 CSV 加载到歌手表中。
*   [load-update-data-Singers . YAML](https://github.com/cloudspannerecosystem/liquibase-spanner/blob/master/example/load-update-data-singers.yaml):从 CSV 插入/更新歌手表。
*   [add-lookup-table-singers-countries . YAML](https://github.com/cloudspannerecosystem/liquibase-spanner/blob/master/example/add-lookup-table-singers-countries.yaml):创建 Countries 表，作为 Singers 中 Country 字段的外键。
*   [修改-数据-类型-歌手-姓氏. yaml](https://github.com/cloudspannerecosystem/liquibase-spanner/blob/master/example/modify-data-type-singers-lastname.yaml) :修改歌手姓氏列中的数据类型。
*   [insert.yaml](https://github.com/cloudspannerecosystem/liquibase-spanner/blob/master/example/insert.yaml) :在歌手表中插入行。
*   [delete.yaml](https://github.com/cloudspannerecosystem/liquibase-spanner/blob/master/example/delete.yaml) :删除歌手的行。
*   [update.yaml](https://github.com/cloudspannerecosystem/liquibase-spanner/blob/master/example/update.yaml) :更新歌手中的行。

# 限制

应该考虑几个[限制](https://github.com/cloudspannerecosystem/liquibase-spanner#limitations)。

# 信用

[Knut lite](https://github.com/olavloite)、 [Andrew James](https://github.com/andrew-james-dmw) (Credera UK)负责原始 Liquibase 代码，以及 [Stefan Serban](https://github.com/sserban-us) 。