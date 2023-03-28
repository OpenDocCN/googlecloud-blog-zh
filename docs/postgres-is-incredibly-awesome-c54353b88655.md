# Postgres 真是太棒了

> 原文：<https://medium.com/google-cloud/postgres-is-incredibly-awesome-c54353b88655?source=collection_archive---------0----------------------->

我怀疑很大比例的用户是为了 PostGIS 而来到 Postgres 的。您可以获得包装在熟悉的 SQL 语法中的空间数据和操作。这使得熟悉关系数据库的开发人员能够快速开始处理空间数据。PostGIS 与其他流行的开源工具，如 GDAL (OGR)、GeoServer 和 QGIS，以及 ESRI 和 Safe Software (FME)的商业产品配合良好；仅举几个例子。另外，Django ORM 可以处理[空间数据和操作](https://docs.djangoproject.com/en/1.10/ref/contrib/gis/),这使得开发人员能够进行一些严肃的地理空间处理。

也就是说，*还有其他不应该被忽视的特点。特别是:交叉表、数组列和内置 JSON 支持。而 Postgres 首先是一个*关系数据库*；所有这些特性，以它们自己的方式，允许设计者在某种程度上反规范化数据库。*

## [交叉表](https://www.postgresql.org/docs/9.6/static/tablefunc.html)

*tablefunc* 模块包括各种返回表格的函数。特别是，*交叉表*函数:

> 生成包含行名和 N 个值列的“数据透视表”，其中 N 由调用查询中指定的行类型决定。

如果这个没有太大意义的话，不用担心……从这个简单的解释中也不是很清楚这是什么意思。与其在这里详述，不如看看这篇[博文](http://imasdemase.com/en/programacion-2/tablas-cruzadas-en-postgresql-pivotmytable/)中的实用解释和出色的助手功能。

## [数组](https://www.postgresql.org/docs/9.6/static/arrays.html)

> PostgreSQL 允许将表中的列定义为可变长度的多维数组。可以创建任何内置或用户定义的基类型、枚举类型或复合类型的数组。

此功能允许设计人员创建可存储可变数据量的列；例如，通常可以用通过附加表定义的多对多关系来处理的事情。数组列可以被索引，因此查找重叠行的查询可以非常快。例如，对“测试”项目中的所有“雇员”进行计数可能如下所示:

```
SELECT COUNT(*) FROM employees WHERE projects && ‘{test}’;
```

而不是这个:

```
SELECT COUNT(*)
FROM 
 employees, 
 employees_projects,
 projects
WHERE 
 employees.id = employees_projects.employee_id AND
 employees_projects.project_id = projects.id AND
 projects.name = 'test';
```

## [JSON](https://www.postgresql.org/docs/9.6/static/datatype-json.html)

> JSON 数据类型用于存储 JSON (JavaScript 对象表示法)数据，如 [RFC 7159](http://rfc7159.net/rfc7159) 中所规定。这种数据也可以存储为文本，但是 JSON 数据类型的优势在于，根据 JSON 规则，每个存储的值都是有效的。还有各种特定于 JSON 的函数和操作符可用于存储在这些数据类型中的数据；参见[第 9.15 节](https://www.postgresql.org/docs/9.6/static/functions-json.html)。

JSON 列也可以用 GIN 索引，这使得“存在和包含”查询非常有效。这种方法的一个实际应用是在同一个表中存储“相似”数据的能力。也就是说，公共字段可以提取到常规列中(带索引等)。)其余的存储在一个包罗万象的 JSON 列中。在极端情况下，您可以制作一个只有一列的表，每行包含一个 JSON 文档……就像穷人的 MongoDB 一样？

Postgres(或任何关系数据库)的一个缺点是，集中式数据库可能很快成为分布式系统中的瓶颈。解决这个问题的标准方法是设置复制。这为您提供了只读副本，可用于处理传入的请求，并在主服务器出现故障或损坏时进行故障转移替换。这里有一个很好的关于如何在谷歌云上设置它的教程:

 [## 如何设置 PostgreSQL 实现高可用性和带热备用的复制| Google Cloud…

### 从社区提交的谷歌云平台社区教程不代表官方谷歌云平台…

cloud.google.com](https://cloud.google.com/community/tutorials/setting-up-postgres-hot-standby) 

所以，[我做了那个](https://twitter.com/aliasmrchips/status/757020598195191808)，它*非常棒……直到待机开始落后。这不是一个生产系统，所以“没有伤害，没有犯规”。最后，我选择不花费资源来让它工作。除了正确配置之外，我们还需要插入一个连接池/负载平衡器来跨副本分配读取，开发一个故障转移和恢复过程，等等。在我们发展的这个阶段，我们有机会安全地做出这个决定，但情况并非总是如此。这是对今年早些时候 GitLab 宕机的一次出色而深思熟虑的事后分析，很好地说明了 Postgres 复制的潜在缺陷。*

[](https://about.gitlab.com/2017/02/10/postmortem-of-database-outage-of-january-31/) [## 1 月 31 日数据库中断的事后分析

### 2017 年 1 月 31 日，我们的产品之一在线服务 GitLab.com 经历了一次重大服务中断。的…

about.gitlab.com](https://about.gitlab.com/2017/02/10/postmortem-of-database-outage-of-january-31/) 

运营一家初创公司的最大挑战之一是决定*到* *做什么*以及*不做什么* *做什么。*高层次确实如此；也就是，“[我们在解决什么问题？我们在为谁解决？我们如何衡量成功？](https://twitter.com/aliasmrchips/status/814704792945696768)“专注很重要。这对我们的使命和我们的技术选择都是正确的。我们必须找到一条从科学到产品的道路，而不是重新发明轮子。这也是我们乐于为 Google 云平台发布 Postgres 云 SQL 提供案例研究的原因之一。

[](https://cloudplatform.googleblog.com/2017/03/Cloud-SQL-for-PostgreSQL-managed-PostgreSQL-for-your-mobile-and-geospatial-applications-in-Google-Cloud.html) [## 云 SQL for PostgreSQL:在 Google 中为您的移动和地理空间应用托管 PostgreSQL…

### 在 Google Cloud Next’17 上，我们宣布支持 PostgreSQL，作为我们托管数据库服务 Google Cloud SQL 的一部分…

cloudplatform.googleblog.com](https://cloudplatform.googleblog.com/2017/03/Cloud-SQL-for-PostgreSQL-managed-PostgreSQL-for-your-mobile-and-geospatial-applications-in-Google-Cloud.html) 

云 SQL for PostgreSQL 是一个全面管理的数据库解决方案，具有灵活的备份和自动存储增加，包括标准；高可用性功能“即将推出”。厉害！最后，如果你错过了，这里是谷歌总理 Brett Hesterberg 和我的同事笛卡尔实验室联合创始人 Tim Kelton 在 Google Cloud Next’17 上谈论云 SQL 和我们的最新产品“GeoVisual Search ”: