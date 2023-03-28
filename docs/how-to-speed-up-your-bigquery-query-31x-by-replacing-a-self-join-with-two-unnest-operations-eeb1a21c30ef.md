# 如何通过用两个 UNNEST()操作替换自连接来加快 BigQuery 查询 31x 的速度

> 原文：<https://medium.com/google-cloud/how-to-speed-up-your-bigquery-query-31x-by-replacing-a-self-join-with-two-unnest-operations-eeb1a21c30ef?source=collection_archive---------0----------------------->

*本故事最初发表于*[*datan coff . ee*](https://datancoff.ee/2017/07/how-to-speed-up-your-bigquery-query-31x-by-replacing-a-self-join-with-two-unnest-operations/)*。*

我在[意见分析项目](https://github.com/GoogleCloudPlatform/dataflow-opinion-analysis)中的一个趋势计算[查询](https://github.com/GoogleCloudPlatform/dataflow-opinion-analysis/blob/master/src/main/java/com/google/cloud/dataflow/examples/opinionanalysis/StatsCalcPipelineUtils.java)最近开始惹麻烦了。它将运行 270 秒，并因错误“查询超出了第 1 层的资源限制”而中断。需要第 8 级或更高级别。”将计费层更改为推荐的第 8 层确实有所帮助，但是查询仍然需要 380 秒才能完成。经过一些研究后，我发现我在部分查询中使用的自连接是罪魁祸首。看一下这个查询底部的连接条件(在 CalcStatCombiTopics 临时表中):

```
INSERT INTO opinions.stattopic (...)
WITH 
p AS (
 SELECT 20170630 AS SnapshotDateId
),
CalcStatSentiments AS (
 SELECT p.SnapshotDateId, t.Tag, ... s.SentimentHash,...
 FROM opinions.document d, p
 INNER JOIN opinions.sentiment s ON s.DocumentHash = d.DocumentHash, UNNEST(s.Tags) AS t
 INNER JOIN opinions.webresource wrOrig ON wrOrig.DocumentHash = d.DocumentHash
 INNER JOIN opinions.webresource wrRepost ON wrRepost.DocumentCollectionId = d.DocumentCollectionId
 AND wrRepost.CollectionItemId = d.CollectionItemId
 WHERE
 d.PublicationDateId = p.SnapshotDateId AND s.SentimentTotalScore > 0
),
CalcStatTopics AS (
…
),
CalcStatCombiTopics AS (
 SELECT 
 css1.SnapshotDateId, CONCAT(css1.Tag,’ & ‘,css2.Tag) AS Topic, [css1.Tag,css2.Tag] AS Tags, true AS GoodAsTopic, 2 AS TagCount,
 ...
 FROM
 CalcStatSentiments css1, CalcStatSentiments css2
 WHERE
 css1.SentimentHash = css2.SentimentHash AND
 css1.Tag < css2.Tag
 GROUP BY css1.SnapshotDateId, css1.Tag, css2.Tag
),
…
```

基本上，我有一个表“opinions . perspective ”,它有一个标识列 SentimentHash、一个重复的字段标签和一堆其他列。Tags 列包含一个文本标记数组，我使用意见分析[云数据流](https://cloud.google.com/dataflow) [IndexerPipeline](https://github.com/GoogleCloudPlatform/dataflow-opinion-analysis/blob/master/src/main/java/com/google/cloud/dataflow/examples/opinionanalysis/IndexerPipeline.java) 从文本中提取这些标记。在 BigQuery 出现之前，我会使用一个单独的表来存储标签，并通过 identity 列 SentimentHash 将其链接到主情感表。然而，在 BigQuery 中，对于重复的字段，这要容易得多。

当我计算趋势时，我为标签的组合建立频率统计(例如，有多少新闻文章是关于“气候变化”和“G-20”的)。为此，我展平了 Tags 字段，并创建了一个临时表 CalcStatSentiments，其中包含每个标记的单独记录，以及 SentimentHash 和实际标记等字段。然后，我对 CalcStatSentiments 表进行自连接，以构建我所谓的“主题”(在 CalcStatCombiTopics 临时表中)。

事实证明，自连接不利于(您的健康)查询的性能，正如这篇[博客文章](http://www.lunametrics.com/blog/2016/05/12/self-joins-windowing-user-defined-functions-bigquery/)中所阐述的。它建议用窗口来代替自连接。我考虑过这样做，但实际上我需要以标记的排列而不是聚集统计来结束，对于聚集统计来说，窗口会工作得很好，所以我想出了另一种技术。

**宣布谢尔盖的自加入淘汰技术**

我不再仅仅展平我的 Tags 字段一次(在我的查询的 CalcStatSentiments 部分)， ***我现在第一次展平它，在结果集中携带 Tags 数组，然后第二次展平它以模拟交叉连接操作。***

下面是它在新查询中的样子:

```
WITH 
p AS (
 SELECT 20170630 AS SnapshotDateId
),
SentimentTags AS (
  SELECT p.SnapshotDateId, s.SentimentHash, t.Tag, t.GoodAsTopic, s.Tags AS Tags
  FROM p, opinions.sentiment s, UNNEST(s.Tags) AS t
  WHERE
    s.DocumentDateId = p.SnapshotDateId AND s.SentimentTotalScore > 0
),
SentimentTagCombos AS (
  SELECT st.SnapshotDateId, st.SentimentHash, st.Tag AS Tag1, stt.Tag AS Tag2 
  FROM SentimentTags st, UNNEST(st.Tags) stt
  WHERE st.Tag < stt.Tag
),
...
```

不等式过滤器` WHERE st.Tag < stt.Tag` ensures that I do not get duplicates in my tag combos. It works the same way as the inequality filter in the original version of my query.

```
WHERE
 css1.SentimentHash = css2.SentimentHash AND
 css1.Tag < css2.Tag
```

Once I calculated my Tag1 & Tag2 combinations, I join my result set via the SentimentHash record ID to my main dataset and conclude all the calculations.

The result: instead of a query that takes 380 seconds to complete in billing tier 8, my modified query runs in 12 seconds in billing tier 1.

Conclusion:

Using self-joins = BAD.
替换为 dual UNNEST() =无价！(或者类似的东西)

如果您想比较语法，下面是一些查询。

新的性能优化查询:

原始查询: