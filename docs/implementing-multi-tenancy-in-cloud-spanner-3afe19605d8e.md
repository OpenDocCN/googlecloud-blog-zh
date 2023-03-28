# 在云扳手中实现多租户

> 原文：<https://medium.com/google-cloud/implementing-multi-tenancy-in-cloud-spanner-3afe19605d8e?source=collection_archive---------1----------------------->

## 克里斯托弗·巴斯勒和西里莎·普利帕蒂

# 介绍

多租户是一种软件架构模式，其中一个或几个应用程序实例服务于多个租户或客户，通常有数百或数千个。这种方法是云计算平台的基础，在云计算平台中，底层基础设施由多个组织共享。基本上，多租户可以被认为是一种基于共享计算资源(如数据库)的分区形式。一个类比是公寓大楼中的租户:共享基础设施，但专用租户空间。多租户也是大多数软件即服务(SaaS)应用程序的标志。

一个例子是一个人力资源 SaaS 提供商在 Google Cloud 上实现其多租户应用程序。多租户应用程序由人力资源 SaaS 提供商的多个客户访问。在多租户应用程序的上下文中，这些客户被称为租户。在本文中，术语“租户”、“客户”或“组织”可以互换使用，以表示访问多租户应用程序的实体。

在多租户应用程序中，每个租户的数据都以底层数据库中几种架构方法中的一种进行隔离。 [Cloud Spanner](https://cloud.google.com/spanner) 是 Google Cloud 的完全托管、企业级、分布式和高度一致的数据库，结合了关系数据库模型的优势和非关系水平可伸缩性。Cloud Spanner 具有带模式的关系语义、强制数据类型、强一致性和多语句 ACID 事务，以及实现 ANSI 2011 SQL 的 SQL 查询语言。

Cloud Spanner 以高达 5 个 9 的可用性 SLA 为计划内维护或区域故障提供零停机时间。Cloud Spanner 提供高可用性和可伸缩性的能力使其成为现代多租户应用程序的一个有吸引力的选择。在本文中，我们将讨论使用 Cloud Spanner 实现多租户的不同架构方法。

# 标准类别

如何将租户数据映射到 Cloud Spanner 的不同架构方法如下:

*   **实例**。一个租户专门驻留在一个 Cloud Spanner 实例中，该租户只有一个数据库。
*   **数据库**。一个租户驻留在一个数据库中，多个数据库位于一个云扳手实例中。
*   **模式**。一个租户驻留在数据库中的专用表中，几个租户可以位于同一个数据库中。
*   **表**。租户数据是与其他租户共享的表中的行。

这些被称为数据管理模式，将在下面详细讨论。他们的讨论基于以下标准:

*   **隔离**。跨多个租户的数据隔离程度是一个主要考虑因素。它是由对其他类别下的标准所做的选择驱动的。例如，某些法规和合规性要求可能要求更高程度的隔离。
*   **敏捷**。租户 wrt 的登船和离船活动的便利性。实例、数据库或表的创建。
*   **作战**。实施典型的特定于租户的数据库操作和管理活动(如定期维护、日志记录、备份或灾难恢复操作)的可用性或复杂性。
*   **音阶**。能够无缝扩展，以适应租户规模和数量的未来增长。在对每个合作伙伴的描述中，讨论了模式可以支持的租户数量。
*   **性能**。能够为每个租户分配专有资源，解决[噪音邻居](https://en.wikipedia.org/wiki/Cloud_computing_issues#Performance_interference_and_noisy_neighbors)现象，并为每个租户提供可预测的读/写性能。
*   **成本**。与所选数据管理模式相关的成本，以及在租户之间按比例分摊成本的能力(如果需要)。
*   **法规与合规**。满足高度管控行业和国家要求的功能，这些行业和国家可能需要完全隔离资源和维护操作。例如，法国的数据居住要求要求个人身份信息(PII)完全存储在法国境内。

下一节将详细描述每种数据管理模式，因为它们与这些标准相关。对于给定的一组租户，可以使用相同的标准来选择使用哪种模式。

# 数据管理模式

以下部分描述了多租户环境中的四种主要数据管理模式:实例、数据库、模式和表。

## 实例—每个实例一个租户

在这种数据管理模式中，每个租户的数据都存储在自己的 Cloud Spanner 实例和数据库中。一个 Cloud Spanner 实例可以有一个或多个数据库，但是，在这种模式下，为了完全隔离，在一个实例中只创建一个数据库。以上面的 HR 应用程序为例，为每个客户组织创建一个单独的 Cloud Spanner 实例，每个组织有一个数据库。

![](img/4956e59480223b8912650773f933884a.png)

数据管理模式:每个实例一个租户

为每个租户提供单独的实例允许使用单独的 Google Cloud 项目来为不同的租户实现单独的信任边界。另一个好处是，可以根据每个租户的位置选择区域性或多区域性的每个实例配置，从而优化成本和性能。

这种数据管理模式提供了租户数据与其他租户数据的完全隔离。该架构可以针对任意数量的租户轻松扩展，因为 SaaS 提供商可以在所需区域创建任意数量的实例，而没有任何实际的硬性限制。下表描述了这种数据管理模式在上述不同标准下的表现。

```
**Criteria** | **"Instance — One tenant per instance"** | **data management pattern** -------------+------------------------------------------------------ **Isolation**    | Enables greatest level of isolation; no database 
             | resources are shared among tenants.
-------------+------------------------------------------------------**Agility**      | Onboarding and offboarding requires considerable 
             | effort to set up or decommission the Cloud Spanner 
             | instance, instance specific security, and 
             | connectivity. This can be automated through 
             | [Infrastructure as Code](https://cloud.google.com/solutions/infrastructure-as-code).
-------------+------------------------------------------------------**Operations** |Backups can be performed independently for each 
             | tenant, providing separation and flexibility in 
             | backup schedules. This data management pattern 
             | results in higher operational overhead owing to the 
             | large number of instances to manage and maintain with 
             | respect to scaling, monitoring, logging, and 
             | performance tuning.
-------------+------------------------------------------------------**Scale** |High scalability, both with respect to the number of 
             | customers as well as tenant specific scalability. 
             | Cloud Spanner, being a highly scalable database, can 
             | accommodate virtually unlimited growth for a tenant 
             | by increasing the number of nodes. This pattern can 
             | support an unlimited number of tenants as for each 
             | tenant a Cloud Spanner instance can be created.
-------------+------------------------------------------------------**Performance** |There is no resource contention among different 
             | tenants owing to separate instances. Also allows for 
             | tailored performance tuning and troubleshooting for a 
             | tenant without impacting others.
-------------+------------------------------------------------------**Cost** |Cloud Spanner is priced based on the number of nodes, 
             | amount of storage and amount of network used and not 
             | based on the number of instances. That means, using a 
             | single instance with a higher number of nodes vs 
             | multiple instances with a smaller number of nodes 
             | each totaling to the same number of nodes effectively 
             | amounts to the same cost. This data management 
             | pattern does not necessarily result in higher cost 
             | compared to other data management patterns in that 
             | aspect. However, isolated instances cannot share 
             | resources and thereby cannot attain resource 
             | efficiencies that may be possible with other data 
             | management patterns for applicable workload patterns.
-------------+------------------------------------------------------**Regulatory** |In this data management pattern it is possible to **and** |store data in specific regions, implement specific **compliance** |security, backup or auditing processes as per the **requirements** |regulatory and compliance requirements applicable for 
             | some industries.
```

总而言之，关键要点是:

*   优势:最高水平的隔离
*   缺点:最大的运营开销

这种数据管理模式最适合下列情况:

*   不同的租户分布在广泛的地区，需要本地化的解决方案
*   一些租户的法规和合规性要求需要更高级别的安全性和审计协议
*   租户的规模变化很大，因此在高容量、高流量租户之间共享资源可能会导致争用和相互降级

## 数据库—每个数据库一个租户

在这种数据管理模式中，每个租户驻留在单个 Cloud Spanner 实例中的数据库中。如果一个实例不足以满足租户的数量，可以创建多个实例。这意味着单个 Cloud Spanner 实例在多个租户之间共享，因为在该模式的上下文中，多个数据库可以驻留在单个实例中。

Cloud Spanner 有一个每个实例 100 个数据库的[硬限制](https://cloud.google.com/spanner/quotas#database_limits)。这意味着，如果 SaaS 提供商需要扩展到超过 100 个客户，就需要创建和使用多个云扳手实例。

在 HR 应用程序的例子中，SaaS 提供者在一个 Cloud Spanner 实例中用一个单独的数据库创建和管理每个租户。

![](img/ed5f58269e18f488009755201f542ea0.png)

数据管理模式:每个数据库一个租户

这种数据管理模式为不同租户的数据实现了数据库级的逻辑隔离。然而，由于它是单个云扳手实例，所有租户数据库共享相同的区域配置和底层计算和存储设置。下表描述了这种数据管理模式在我们的标准下的表现。

```
**Criteria** | **"Instance — One tenant per database"** | **data management pattern** -------------+------------------------------------------------------ **Isolation**    | Complete logical isolation on a database level; 
             | underlying instance infrastructure resources are 
             | shared.
-------------+------------------------------------------------------**Agility**      | Onboarding and offboarding requires some effort to 
             | create or delete the database and any specific 
             | security controls. Can rely on automation through 
             | Infrastructure as Code.
-------------+------------------------------------------------------**Operations** |Backups can be performed independently for each 
             | tenant providing flexibility in schedules. This data 
             | management pattern involves less operational overhead 
             | compared to the instance data management pattern, you 
             | have only one instance to monitor for performance and  
             | scale (for up to 100 databases).
-------------+------------------------------------------------------**Scale** |Cloud Spanner instances can be scaled to thousands of 
             | nodes and can accommodate any level of growth for the 
             | tenants.
             | There is a limit of 100 databases per Cloud Spanner 
             | instance currently which limits the number of tenants 
             | on-boarded onto a single instance. For every 100 
             | tenants a new Cloud Spanner instance needs to be 
             | created and there is no limit to the number of 
             | instances.
-------------+------------------------------------------------------**Performance** |Databases are spread across Cloud Spanner instance 
             | nodes and thus share the infrastructure resulting in
             | resource contention among multiple databases (noisy 
             | neighbor effect) and impacts the performance.
-------------+------------------------------------------------------**Cost** |Cloud Spanner is priced based on the resources 
             | consumed rather than on the number of databases or 
             | instances. So, this data management pattern doesn’t 
             | necessarily result in higher cost compared to others. 
             | However, the shared resources in terms of compute and 
             | storage can result in efficiencies that can reduce 
             | the overall cost. The converse, where severe resource 
             | contention requires additional node capacity than 
             | otherwise, is possible too.
-------------+------------------------------------------------------**Regulatory** |In this data management pattern, it is not possible **and** |to meet any specific data residency regulatory **compliance** |requirements, if the desired location is different **requirements** |from the instance regional configuration.
```

总而言之，关键要点是:

*   优势:隔离级别更高
*   缺点:每个实例的租户数量有限，位置不灵活

这种数据管理模式最适合下列情况:

*   多个客户位于同一地理区域，例如美国，和/或受同一监管机构管辖
*   租户需要基于系统的数据分离和备份/恢复，但对基础架构资源共享没有问题。

## 模式—每个租户一组表

在模式数据管理模式中，实现单个模式的单个数据库用于多个租户，每个租户的数据使用一组单独的表。通过在表名中包含租户 ID 作为后缀或前缀，可以区分这些表。

与上述选项(实例级、数据库级)相比，这种为每个租户使用一组单独的表的数据管理模式提供了低得多的隔离级别。然而，这种方法使 onboarding 变得非常简单，因为它只涉及创建新的表以及相关的引用完整性和索引。然而，一个大的警告是，Cloud Spanner through Cloud IAM 的访问权限只能在实例或数据库级别提供，而不能在表级别提供。每个数据库的表数也有限制——5000，这限制了应用程序对于大量客户的可伸缩性。此外，为每个客户使用单独的表会导致大量模式更新操作积压，需要很长时间才能完成。

对于 HR 应用程序的示例，SaaS 提供程序可以为每个客户创建一组表，在表名中以租户 ID 作为前缀，如下所示

客户 1 _ 员工、客户 1 _ 工资单、客户 1 _ 部门等。

![](img/a5ad9a567fee114c37294cbf5e2f0721.png)

数据管理模式:每个租户一组表

下表概述了这种数据管理模式如何影响不同的标准。

```
**Criteria** | **"Schema — One set of tables for each tenant"** | **data management pattern** -------------+------------------------------------------------------ **Isolation**    | Low level of isolation; no table level security.
-------------+------------------------------------------------------**Agility**      | Onboarding a customer is a fairly simple task 
             | involving creation of new tables and associated keys 
             | and indexes. Offloading a customer means deleting the 
             | tables, which may have a temporary negative impact on 
             | the performance of the other tenants within the 
             | database.
-------------+------------------------------------------------------**Operations** |Operations including backups, monitoring, logging 
             | cannot be performed separately for tenants by Cloud 
             | Spanner itself. Those have to be implemented as 
             | separate functionality by the application itself or 
             | as utility scripts.
-------------+------------------------------------------------------**Scale** |Cloud Spanner instances can be scaled to thousands of 
             | nodes and can accommodate any level of growth for the 
             | tenants. However, there is a limit of 5000 tables a 
             | single database can have.
             | This pattern therefore supports only 
             | floor(5000/<number tables for tenant>) tenants in 
             | each database. If that is exhausted, a new database 
             | needs to be added for additional tenants.
-------------+------------------------------------------------------**Performance** |High level of resource contention is possible (noisy 
             | neighbor). In order to ensure good performance, 
             | indexes need to be designed separately for each set 
             | of tables as well.
-------------+------------------------------------------------------**Cost** |Cloud Spanner is priced based on the resources 
             | consumed rather than on the number of tables, 
             | databases or instances. So, this data management 
             | pattern does not necessarily result in higher cost 
             | compared to others. However, the shared resources in 
             | terms of compute and storage can result in 
             | efficiencies that can reduce the overall cost. The 
             | converse, where severe resource contention requires 
             | additional node capacity than otherwise, is possible 
             | too.
-------------+------------------------------------------------------**Regulatory** |In this data management pattern, it is not possible **and** |to meet any specific data residency regulatory **compliance** |requirements, if the desired location is different **requirements** |from the instance regional configuration. Also
             | implementing specific security and auditing controls 
             | impacts all the tenants that reside in the same 
             | database.
```

总而言之，关键要点是:

*   优势:易于入职
*   缺点:较高的操作开销和缺乏表级别的安全控制

这种数据管理模式最适合下列情况:

*   满足不同部门需求的内部应用程序，在这些部门中，严格的数据安全隔离与易维护性相比不是一个突出的问题。
*   根据法律或法规要求，数据不需要严格分离的多租户应用程序。

**注意:**虽然可以在一个数据库中创建几组表(每组代表一个租户)，但从数据库的角度来看，这是最不理想的模式。主要原因是表必须遵循命名约定，不仅应用程序，而且任何数据库工具(例如 IDE、模式迁移工具)都必须理解这一点。此外，如果每个租户的表数量相当多，这并不能真正提供显著的伸缩性。更好的方法是转移到每个租户一个数据库并增加实例数量，或者转移到表模式。

## 桌子—几个租户共用一张桌子

最终的数据管理模式是用一组公共的表为多个租户服务。每个表包含几个租户的数据。这种数据管理模式代表了一种极端的多租户模式，在这种模式下，从基础设施到模式再到数据模型，一切都在多个租户之间完全共享。在一个表中，行是根据主键进行分区的，租户 ID 是键的第一个元素。从可伸缩性的角度来看，Cloud Spanner 可以最好地支持这种模式，因为它可以无限制地扩展表。

以 HR 应用程序为例，工资单表的主键可以是客户 ID 和工资单 ID 的组合。

![](img/2a57036972ff0d87cbe8a93a17737382.png)

数据管理模式:几个租户一个表

与模式模式类似，不能为不同的租户单独控制数据访问。与每个租户在数据库中拥有自己的表相比，使用更少的表意味着模式更新操作可以更快地完成。这种方法在很大程度上简化了上车、下车活动以及操作。

下表根据考虑中的标准列表评估了这种数据管理模式。

```
**Criteria** | **"Table — One table for several tenants"** | **data management pattern** -------------+------------------------------------------------------ **Isolation**    | Lowest level of isolation; no tenant level security.
-------------+------------------------------------------------------**Agility**      | Onboarding a customer does not require any setup on 
             | the database side. The application can write data 
             | directly into the existing tables. Offboarding simply 
             | means deleting the customer’s rows.
-------------+------------------------------------------------------**Operations** |Operations including backups, monitoring, logging 
             | cannot be performed separately for tenants by Cloud 
             | Spanner and they have to be implemented separately. 
             | Little to no overhead as the number of tenants 
             | increases.
-------------+------------------------------------------------------**Scale** |Cloud Spanner instances can be scaled to thousands of 
             | nodes and can accommodate any level of growth for the 
             | tenants.
             | This pattern can support an unlimited number of 
             | tenants.
-------------+------------------------------------------------------**Performance** |This pattern works very well in the context of Cloud 
             | Spanner as its internal distributed storage and 
             | processing as well as load balancing can easily deal 
             | with this pattern.
             | A high level of resource contention is possible 
             | (noisy neighbor) if the primary key spaces are not 
             | designed carefully preventing concurrency and 
             | distribution, though. Following best practices is 
             | very important. Deleting a tenant’’s data might have 
             | a temporary impact on the load.
-------------+------------------------------------------------------**Cost** |Cloud Spanner is priced based on the resources 
             | consumed rather than on the number of tables, 
             | databases or instances. The shared resources in terms 
             | of compute and storage can result in efficiencies 
             | that can reduce the overall cost. The converse, where 
             | severe resource contention requires additional node 
             | capacity than otherwise, is possible too.
-------------+------------------------------------------------------**Regulatory** |In this data management pattern, it is not possible **and** |to meet any specific data residency regulatory **compliance** |requirements, if the desired location is different **requirements** |from the instance regional configuration. It cannot
             | provide system level partitioning either (compared to 
             | the instance or database pattern). Also any specific 
             | security and auditing controls cannot be implemented 
             | without affecting all the tenants.
```

总而言之，关键要点是:

*   优势:高可伸缩性和低运营开销
*   缺点:高资源争用，缺乏对每个租户的安全控制

这种模式最适合下列情况:

*   满足不同部门需求的内部应用程序，在这些部门中，严格的数据安全隔离与易维护性相比不是一个突出的问题。
*   使用免费层应用程序功能的租户可最大限度地共享资源，同时最大限度地降低成本

# 数据管理模式和租户生命周期管理的结合

## 数据管理模式概述

下表比较了所有标准中的各种数据管理模式，以便从上面的细节中提供一个非常高层次的概述。

![](img/8fdc4566ed6fec47dbbe75f316a03c97.png)

*性能在很大程度上取决于[模式设计](https://cloud.google.com/spanner/docs/schema-design)和[查询最佳实践](https://cloud.google.com/spanner/docs/sql-best-practices)，因此这里的值只是平均期望值。
**成本基于分配给云扳手实例的节点数量。不同模式之间的直接比较很困难，因为相同数量的租户可能会导致不同数量的节点，这取决于查询访问模式、事务频率、租户访问并发性和模式设计(由于资源争用的存在和级别)。我们强烈建议在最终设计决策之前，在概念验证中测试不同的模式。

特定多租户应用程序的最佳数据管理模式是那些根据标准满足其大部分需求的模式。如果应用程序不需要特定的标准，则相应的行在决策过程中不起作用。

## 数据管理模式的组合

在许多情况下，单个数据管理模式足以满足多租户应用程序的需求。多租户应用程序的设计可以采用单一的数据管理模式，所有与 Cloud Spanner 的交互都基于实现的数据管理模式。

然而，一些多租户应用程序同时需要几种数据管理模式。一个很好的例子是一个多租户应用程序，它为其客户端支持免费层、常规层和企业层。

*   **自由层**。免费层必须具有成本效益，并且具有例如所支持的数据量的上限。它通常也支持有限的功能。表数据管理模式是免费层的一个很好的候选模式，因为租户管理很简单，不需要为该租户创建特定的或独占的资源。
*   **普通级**。对于在伸缩性或隔离性方面没有特别强烈要求的付费客户，可以建立常规层。对于这些客户机，模式数据管理模式或数据库数据管理模式可能是合适的。在这两种情况下，表和索引都是租户专用的。不同之处在于，在数据库数据管理模式的情况下，备份很容易，但对于模式数据管理模式，云扳手不支持备份——在这种情况下，租户备份必须作为云扳手外部的实用程序来实现。
*   **企业层级**。企业层通常是高端层，在所有方面都有完全的自主权。该层中的租户拥有专用资源，包括专用扩展和完全隔离。实例数据管理模式非常适合这一需求。

注意，最佳实践是将不同的数据管理模式分离到不同的数据库中。例如，模式和表数据管理模式最好不要在单个数据库中实现。虽然可以在一个数据库中组合不同的数据管理模式，但是应用程序的访问逻辑以及生命周期操作更难以实现。

下面的“应用程序设计”一节将概述一些在使用单个数据管理模式或多个数据管理模式时适用的多租户应用程序设计注意事项。

## 租户数据生命周期管理

租户有一个生命周期，相应的管理操作必须在多租户应用程序中实现。除了创建、更新和删除租户的基本操作之外，还应考虑以下与数据相关的附加操作:

*   **导出租户数据**。当要删除租户时，可能需要首先导出租户数据，并可能使数据集对租户可用。根据数据管理模式，导出必须由多租户应用系统实现(例如，表、模式模式)，或者可以映射到数据库功能(例如，数据库导出)。
*   **备份租户数据**。与导出类似，可能需要备份单个租户的数据。当数据管理模式是实例或数据库时，可以使用导出或备份等数据库功能。然而，在模式或表数据管理模式的情况下，这个操作必须由多租户应用程序实现，因为数据库功能不能确定哪个数据属于哪个租户。
*   **移动租户数据**。一个非常重要的操作是将租户从一种数据管理模式转移到另一种模式(或者在实例或数据库之间的同一数据管理模式中)。用例是表数据管理模式中的一个小租户，它发展到一定程度，必须转移到数据库数据管理模式。这需要从表数据管理模式中提取数据，并将数据插入到数据库数据管理模式中。当应用程序可能停机时，可以执行导出/导入(见上文)，但是，当停机不可能时，这对应于云扳手环境中的[零停机数据库迁移。搬迁租户的另一个原因是为了缓解邻居吵闹的局面。](/google-cloud/zero-downtime-database-migration-and-replication-to-and-from-cloud-spanner-99ad0c654d12)

# 应用设计

多租户应用程序设计是关于实现租户感知的业务逻辑，这意味着业务逻辑的每次执行必须总是在已知租户的上下文中进行。

从数据库的角度来看，这意味着每个查询都必须根据租户所在的数据管理模式来执行。下面的讨论强调了多租户应用程序设计的一些核心概念。

## 动态租户连接和查询配置

映射配置通常用于将租户的数据动态映射到来自租户应用程序的请求。

*   对于实例或数据库数据管理模式，一个连接字符串就足以访问租户的数据。
*   对于模式数据管理模式，必须确定正确的表名。
*   在表数据管理模式的情况下，必须使用适当的谓词对数据库执行查询，以便只检索有问题的租户的数据。

原则上，租户可以驻留在四种数据管理模式中的任何一种。下面的映射实现解决了多租户应用程序同时使用所有数据管理模式的一般情况下的连接配置。给定的租户以一种模式存在。在一些多租户应用程序中，只有一种数据管理模式用于所有租户。这种情况隐含在下面的映射中。

如果租户执行业务逻辑(例如，员工使用其租户 id 登录)，那么应用程序逻辑必须确定租户的数据管理模式、给定租户 id 的数据位置，以及可选的表命名约定(对于模式模式)。

这需要租户到数据管理模式的映射，如下所示:

```
tenant id -> (data management pattern, 
              database connection string, 
              [table_prefix])
```

连接字符串是指租户数据所在的数据库。这标识了云扳手实例以及数据库实例。对于数据管理模式实例和数据库，这足以让应用程序连接和执行查询。

但是，对于模式和表数据管理模式，还需要额外的设计。对于模式数据管理模式，同一个数据库中有几个租户，每个租户都有自己的一组表。这些表通过它们的名称来区分，以便确定哪个表属于给定的租户。

一种方法是在表名前面加上租户 id。例如，对于 id 为`356`的租户，表`EMPLOYEE`被称为`T356_EMPLOYEE`。在将查询发送到映射返回的数据库之前，应用程序必须为每个表添加前缀`T<tenant id>`。

前面的映射中显示了一种替代方法。不使用默认的命名约定，而是将一个表前缀添加到映射中，该表前缀需要在查询中添加到表名之前，以便为租户找到正确的表。

当然，混合方法也是可能的:如果模式是 schema，而表前缀是空的，则使用默认映射。

表数据管理模式也需要类似的设计。但是，在这种情况下，只有一个模式，租户的数据存储为行。为了正确地访问数据，必须在每个选择适当租户的查询后添加一个谓词。

一种方法是在每个表中有一个名为`TENANT`的列，这些列的值是租户 id。每个查询都必须将一个谓词`AND TENANT = <tenant id>`附加到一个现有的`WHERE`子句，或者为此添加一个`WHERE`子句。

要实现与数据库的连接并创建正确的查询，租户标识符必须在应用程序逻辑中可用(可能作为参数传入或作为线程上下文存储)。

一些生命周期操作需要修改租户到数据管理模式的映射配置。例如，当租户在数据管理模式之间移动时，必须更新数据管理模式和数据库连接字符串，可能还需要更新表前缀。

## 查询生成和属性

多租户应用程序的一个基本原则是能够为租户共享资源，以便几个租户可以共享单个云资源。除了将单个租户分配给单个 Cloud Spanner 实例的情况之外，上面的数据管理模式都属于这一类。

资源的共享超越了数据方面。监控和日志记录也是共享的。例如，在表数据管理模式的情况下，所有租户的所有查询都记录在同一个审计日志中。模式数据管理模式也是如此。

如果记录了一个查询，那么必须检查查询文本，以确定执行该查询的租户。在表数据管理模式的情况下，必须解析谓词，在模式数据管理模式的情况下，必须解析一个表名。

为了简化对日志和查询的分析，如果能够在不解析查询文本的情况下确定给定查询的租户，将会更容易。在数据库或实例数据管理模式的上下文中，查询文本根本没有租户信息——必须查询租户到数据管理模式的映射表才能获得该信息。

为了能够在所有数据管理模式中统一地标识查询的租户，一种解决方案是向具有租户 id 的查询文本添加注释，可能还有一个标签。举个例子，

```
select * from EMPLOYEE
-- TENANT 356
where TENANT = 'T356';
```

或者

```
select * from T356_EMPLOYEE;
-- TENANT 356
```

使用这种设计，为租户执行的每个查询都属于该租户，与所使用的数据管理模式无关。如果租户从一种数据管理模式转移到另一种模式，查询文本可能会改变，但是属性将是查询文本中完全相同的注释。

以上只是一个例子。代替标签和值， [JSON](https://www.json.org/) 对象也可以作为注释插入:

```
select * from T356_EMPLOYEE;
-- {"TENANT": 356}
```

## 租户访问生命周期操作

根据设计理念，多租户应用程序可以直接实现前面描述的数据生命周期操作，也可以创建单独的租户管理工具。

独立于数据生命周期操作的实现策略，一个重要的方面是生命周期操作可能必须在没有应用程序逻辑同时执行的情况下独占执行。例如，在将租户从一种数据管理模式转移到另一种模式时，应用程序逻辑无法执行，因为数据不在单个数据库中。从应用的角度来看，这需要两个额外的操作:

*   **停止租户**。此生命周期操作禁用所有应用程序逻辑访问，同时允许数据生命周期操作。
*   **启动租户**。这个生命周期操作实现了相反的约束:应用程序逻辑可以访问租户的数据，而那些会干扰应用程序逻辑的生命周期操作被禁用。

虽然不经常使用，但紧急租户关闭可能是另一个重要的生命周期操作，当需要禁止对租户数据的所有访问时，不仅是应用程序逻辑，还有生命周期操作，以防可疑违规。违规可能来自外部，也可能来自内部。

移除紧急状态的匹配生命周期操作也必须可用，可能需要两个或更多管理员同时登录，以便实现相互控制。

## 应用隔离

各种数据管理模式支持不同程度的租户数据隔离。从最高隔离级别(实例)到最低隔离级别(表),可能有不同的隔离级别。

在多租户应用程序的上下文中，必须做出类似的部署决策:是否所有租户都使用相同的应用程序部署来访问他们的数据(以可能不同的数据管理模式)？例如，一个 Kubernetes 集群可以支持所有租户，当一个租户访问其数据时，同一个集群正在执行业务逻辑。

或者，与数据管理模式的情况一样，可以将不同的租户定向到不同的应用程序部署。大型租户可以访问专属于他们的应用程序部署，而较小的租户或免费层中的租户共享一个应用程序部署。

尽管将(数据)数据管理模式与等效的应用程序数据管理模式直接匹配很有吸引力，但并不一定非要这样。数据库数据管理模式和所有这些租户共享一个应用程序部署是可能的。

# 摘要

多租户是一种重要的应用程序设计数据管理模式，尤其是当资源效率起着至关重要的作用时。Cloud Spanner 支持多种数据管理模式，可以轻松用于实现多租户应用程序。凭借 Cloud Spanner 极致的可扩展性和超严格的 SLA，是大型多租户应用部署的理想数据库。

# 放弃

Christoph Bussler 是谷歌公司(Google Cloud)的解决方案架构师，Sireesha Pulipati 是数据管理客户工程师。这里陈述的观点是我们自己的，而不是谷歌公司的。