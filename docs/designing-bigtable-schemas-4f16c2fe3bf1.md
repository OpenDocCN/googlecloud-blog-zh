# 设计大表模式

> 原文：<https://medium.com/google-cloud/designing-bigtable-schemas-4f16c2fe3bf1?source=collection_archive---------0----------------------->

Bigtable 是谷歌云平台提供的 NoSQL 数据库服务。在 Bigtable 中，每一行都代表一个实体，用一个唯一的行键进行标记。每列存储每行的属性值，列族可用于组织相关的列。在行和列的交叉处，可以有多个单元格，每个单元格代表给定时间戳的数据的不同版本。

为了设计模式以及应该使用哪个行键，我们可以尝试研究以下问题:

*   单个行代表什么？(识别行结构)
*   对此数据最常见的查询是什么？(创建行键)
*   为每一行收集什么值？(标识列限定符)
*   是否有可以分组或组织在一起的相关列？(识别柱族)

创建 Bigtable 模式的最佳实践

*   将具有相似模式的数据存储在同一个表中，而不是存储在不同的表中
*   使用列限定符作为数据，这样就不会重复每行的值
*   组织同一列族中的相关列
*   为柱族选择简短但有意义的名称
*   根据将用于检索数据的查询来设计行键
*   避免以时间戳或连续数字 id 开头的行键，或者导致相关数据不被分组的行键
*   设计行键，以更常见/通用的值开始，以更细粒度的值结束
*   使用人类可读的字符串值在每个行键中存储多个分隔值

**参考文献**

Chang，f .，Dean，j .，Ghemawat，s .，Hsieh，W. C .，Wallach，D. A .，Burrows，m .，Chandra，t .，Fikes，a .，& Gruber，R. E. (1970 年 1 月 1 日)。 *Bigtable:结构化数据的分布式存储系统*。谷歌研究。检索于 2022 年 8 月 27 日，发自 https://research.google/pubs/pub27898/

谷歌。(未注明)。*模式设计最佳实践|云大表文档|谷歌云*。谷歌。检索于 2022 年 8 月 27 日，来自[https://cloud.google.com/bigtable/docs/schema-design](https://cloud.google.com/bigtable/docs/schema-design#row-keys)