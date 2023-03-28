# Ruby 遇上 BigQuery:第 1 部分

> 原文：<https://medium.com/google-cloud/ruby-meets-bigquery-part-1-d8aa1f9e68f7?source=collection_archive---------5----------------------->

随着我们在 Google 云平台上改进对 Ruby 的支持，我们讨论中出现的一个问题是哪些 gem 被频繁使用。我们希望确保普通的 gems 干净地安装在我们提供的 Ruby 容器上。这意味着在容器映像中包含许多具有本地扩展的 gem 的库。当我们讨论这个问题时，我内心的测试者想知道，最常见的 500 种宝石是什么？幸运的是，有多种数据来源可以解决这个问题。

## Rubygems.org 数据

[Rubygems.org](https://rubygems.org/)每周提供清理过的【PostgreSQL 数据库的转储。转储包括 gem 和版本的下载计数。下载表名为 gem_downloads，模式如下:

```
| Column Name | Type    |
|-------------+---------|
| id          | integer |
| rubygem_id  | integer |
| version_id  | integer |
| count       | bigint  |
```

rubygem_id 和 version_id 是包含 gem 名称、版本字符串和其他相关信息的其他表的外键。

这不是一个特别大的数据集，但是我想把导入的数据放在一个团队中的其他人可以容易地运行特别查询的地方。对我来说，最好的地方是 [BigQuery](https://cloud.google.com/bigquery/) 。BigQuery 是一个托管数据仓库，是谷歌云平台的一部分。它在查询大型数据集时速度惊人，我可以使用标准 SQL 查询它。

# 将数据放入 BigQuery

我使用 rubygems.org 提供的示例加载脚本将数据加载到本地 PostegreSQL 数据库中。从那里将数据加载到 BigQuery 非常简单。对于一些表格，我使用了流方法。下面是加载 gems 表的代码。

```
require 'pg'
require 'gcloud'pg_conn = PG.connect dbname: 'rubygems'gcloud   = Gcloud.new
bigquery = gcloud.bigquery
dataset  = bigquery.dataset "rubygems"bq_table ||= bq_database.create_table("gems") do |s|
 s.integer "id"
 s.string "name"
 s.timestamp "created_at"
 s.timestamp "updated_at"
endrubygems_cols = %w[id name created_at updated_at]pg_conn.exec("SELECT * FROM rubygems") do |result|
  result.each do |row|
    data = Hash[rubygems_cols.zip(row.values)]
    bq_table.insert(data)
  end
end
```

第 21 行有点混乱。PostgreSQL 以值数组的形式返回每一行。BigQuery 需要键值对。zip 和 Hash[]将值数组转换成一个散列，该散列将以正确的格式发送给 BigQuery。

这个例子的其余部分非常简单。首先我需要 pg 和 gcloud 宝石。然后我初始化一个到 BigQuery 和 PostgreSQL 的连接。之后，如果 BigQuery 表不存在，我就创建它。最后，我连接 PostgreSQL，提取数据，并插入到 BigQuery 中。

对于 rubygems.org 数据中的其余表，我使用了批处理。我将表导出到 CSV，然后使用 UI 将它们直接从 CSV 加载到 BigQuery 中。还可以使用 gcloud gem 将 CSV、json 或 avro 文件加载到 BigQuery 中。

# 分析 rubygems.org 数据

一旦数据被加载到 BigQuery 中，分析就像编写 SQL 查询一样简单。我的主要问题是“哪些宝石下载最多？”这个查询获得了下载最多的五个 gem:

```
SELECT name, count 
FROM [rubygems.downloads] 
JOIN rubygems.gems ON rubygems.gems.id = rubygems.downloads.rubygem_id 
ORDER BY count DESC 
LIMIT 5
```

结果如下:

```
| name       | count       |
|------------+-------------|
| rake       | 107,076,261 |
| rack       | 100,955,906 |
| multi_json | 100,171,080 |
| json       | 95,715,131  |
| bundler    | 93,085,862  |
```

我原本预计 Rails 是下载量最大的宝石，但它只排在第 14 位。

gem_downloads 表有一个 gem_id 和一个 version_id 列。下载是针对特定 gem 的特定版本统计的，所以上面的查询没有给出准确的统计。该查询对每个 gem 的所有版本的计数求和。

```
SELECT name, sum(count) as total 
FROM [rubygems.downloads] 
JOIN rubygems.gems ON rubygems.gems.id = rubygems.downloads.rubygem_id 
GROUP BY name 
ORDER BY total DESC 
LIMIT 5
```

这产生了同样的前五名宝石，但具有更高的下载计数。

```
| name       | count       |
|------------+-------------|
| rake       | 214,152,212 |
| rack       | 201,911,759 |
| multi_json | 200,342,260 |
| json       | 191,430,173 |
| bundler    | 186,172,479 |
```

我也很好奇哪个测试库最受欢迎。感觉测试库的讨论就像是 ruby 社区的编辑战争。我非常喜欢 minitest，但许多人仍然使用 rspec。

```
SELECT name, sum(count) as total 
FROM [rubygems.downloads] 
JOIN rubygems.gems 
  ON rubygems.gems.id = rubygems.downloads.rubygem_id 
GROUP BY name 
HAVING name IN ('minitest', 'rspec')| name     | total       |
|----------+-------------|
| minitest | 101,151,246 |
| rspec    | 77,293,803  |
```

Google Cloud Ruby 团队感兴趣的最后一件事是哪个版本的 Rails 最受欢迎。我们都知道有人还在使用旧版本的 Rails 3。为了回答这个问题，我需要获得每个版本 Rails 的下载量。

我们所说的版本号，像 3.1.0，其实就是字符串。为了获得每个版本的计数，我必须提取版本字符串的主要版本部分，忽略次要版本和补丁部分。我使用正则表达式获取版本字符串中第一个点之前的所有内容。

```
SELECT REGEXP_EXTRACT(number,r'(\d)\.') AS major,
  sum(rubygems.downloads.count) AS total 
FROM [rubygems.versions] 
JOIN rubygems.gems 
  ON rubygems.gems.id = rubygems.versions.rubygem_id 
JOIN rubygems.downloads 
  ON rubygems.versions.rubygem_id = rubygems.downloads.rubygem_id WHERE rubygems.gems.name = 'rails' 
GROUP BY name, major 
ORDER BY major| version | downloads              |
|---------+------------------------|
|       0 | 2,890,350,351          |
|       1 | 2,064,535,965          |
|       2 | 3,991,436,199          |
|       3 | 16,378,651,989         |
|       4 | 12,662,487,252         |
|       5 | 963,450,117            |
```

这表明 Rails 3 是下载最多的，Rails 4 也有相当多的下载。

# 衡量下载量的问题

这个数据给了我们一个合理的标准，来衡量哪种宝石最受欢迎。然而，下载量并不一定是衡量使用情况的最佳方式。一个公司有很多服务器，他们经常更新，如果他们每次安装时都从网上下载他们的 gem，可能会严重扭曲数据。我想找到另一种方法来衡量宝石的使用。

当我在写这篇博文的时候，以及随之而来的[谈话](https://www.youtube.com/watch?v=0mzNNg62_kM)。我发现 Google 从 GitHub 发布了近 300 万个开源存储库的信息作为 BigQuery 公共数据集。本系列的下一篇文章将使用这些数据来分析 gem 的受欢迎程度，并比较这两种来源的结果有何不同。

07/13/16

*原载于 2016 年 7 月 13 日*[*【www.thagomizer.com*](http://www.thagomizer.com/blog/2016/07/13/ruby-meets-bigquery-part-one.html)*。*