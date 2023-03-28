# Ruby 遇上 BigQuery:第 2 部分

> 原文：<https://medium.com/google-cloud/ruby-meets-bigquery-part-2-2798b21133a0?source=collection_archive---------6----------------------->

在我之前的文章中，我展示了我如何使用 BigQuery 根据下载量计算出哪些宝石是最受欢迎的。我还展示了如何找出哪个版本的 gems 最受欢迎。但是使用下载数据有一些缺点。一家拥有大量服务器并经常更新的大公司(假设他们不出售 gems 或使用机器映像)很容易扭曲数据。幸运的是，还有另一个数据来源。

## BigQuery 中的 GitHub 数据

最近，谷歌云平台和 GitHub 在 BigQuery 上提供了来自近 300 万个开源存储库的数据。这个数据提供了另一种衡量宝石受欢迎程度的方法。我对此很兴奋，因为这给了我另一种方法来衡量宝石的受欢迎程度。除了查看它被下载的原始次数，我还可以看到有多少项目在他们的 Gemfile 中包含了它(假设他们有一个)。

[GitHub 数据集](https://cloud.google.com/bigquery/public-data/github)很大(超过 3TB)。BigQuery 可以快速查询整个数据集。我的大多数查询不到 30 秒。但是对于每个查询，我都要检查数百万不包含不必要的 Ruby 文件的行。此外，查询整个数据集可能会很昂贵。

为了让我的查询稍微快一点并且便宜很多，我需要限制我的查询，只需要查看 Gemfiles、Rakefiles 和。rb 文件。一种简单的方法是将这些行提取到它们自己的数据集中。

下面是提取所有名为 Gemfile、Gemfile.lock 或 Rakefile 的文件的查询。我把它们放在新数据集中的一个表中。

```
SELECT * 
FROM [bigquery-public-data:github_repos.files] 
WHERE path IN ('Gemfile', 'Gemfile.lock', 'Rakefile')
```

这个查询提取所有的。rb 文件。我使用正确的命令将文件路径的最后三个字符与字符串'进行比较。rb '。

```
SELECT * 
FROM [bigquery-public-data:github_repos.files] 
WHERE RIGHT(path, 3) = '.rb'
```

这些进了另一张桌子。我把宝石锉和耙子锉从。rb 文件因为格式不同，所以需要不同的查询来分析。

## 查询数据集

让我们看看有多少。rb 文件已被提取。

```
SELECT count(*) as num 
FROM [rb_files]|   num    |
|----------|
| 19861839 |
```

所以 GitHub 数据集中有 19，861，839 个以. rb 结尾的文件。gem file 偶尔会被非 Ruby 项目使用(例如 Cocoapods ),所以我预计还有相当数量的 gem file 和 Rakefiles。

```
SELECT count(*) as num 
FROM [gem_rake_contents]|  num   |
|--------|
| 251420 |
```

这些数字并不特别有趣。我想知道最常见的宝石是什么。一种方法是解析 gem 文件并提取 gem 名称。为此，首先我需要将 Gemfile 的内容分成几行，这可以用 split 函数来完成。在我分割了这些行之后，我可以使用 REGEXP_EXTRACT 来提取宝石名称。最后，因为我想包括依赖项和指定的 gem，所以我将只在 Gemfile.lock 上运行这个查询。

```
SELECT REGEXP_EXTRACT(line, r"\s*gem\s['\"](.*?)['\"]") as gem 
FROM ( 
  SELECT SPLIT(content, '\n') as line 
  FROM github_ruby.gem_rake_contents 
) 
HAVING gem IS NOT NULL|     gem       |
|---------------|
| sinatra       |
| mongoid       |
| thin          |
| rake          |
| bcrypt-ruby   |
| rspec         |
| rack-test     |
| mongoid-rspec |
| simplecov     |
| fivemat       |
| factory_girl  |
| timecop       |
```

这给了我们宝石的名字，但它没有给我们任何排序。它只是列出了所有宝石的名称。为了找出最受欢迎的宝石，我需要做一些分组。

```
SELECT gem, COUNT(*) AS n 
FROM ( 
  SELECT REGEXP_EXTRACT(line, r"\s*gem\s['\"](.*?)['\"]") AS gem
  FROM ( 
    SELECT SPLIT(content, '\n') AS line 
    FROM github_ruby.gem_rake_contents 
  ) 
  HAVING gem IS NOT NULL 
) 
GROUP BY gem 
ORDER BY n DESC 
LIMIT 10| gem          |     n |
|--------------+-------|
| rails        | 33227 |
| jquery-rails | 19709 |
| uglifier     | 18369 |
| sass-rails   | 17857 |
| coffee-rails | 16145 |
| rspec        | 15829 |
| rake         | 15683 |
| therubyracer | 14081 |
| pg           | 14029 |
| unicorn      | 13696 |
```

## 结论

使用 Rubygems.org 下载数据，最流行的 gem 是 rake、rack、multi_json、json 和 bundler。

```
| name       | count       |
|------------+-------------|
| rake       | 214,152,212 |
| rack       | 201,911,759 |
| multi_json | 200,342,260 |
| json       | 191,430,173 |
| bundler    | 186,172,479 |
```

使用 GitHub 数据，最流行的 gem 是 rails、jquery-rails、uglifier 和 sass-rails。

```
| gem          |     n |
|--------------+-------|
| rails        | 33227 |
| jquery-rails | 19709 |
| uglifier     | 18369 |
| sass-rails   | 17857 |
| coffee-rails | 16145 |
| rspec        | 15829 |
| rake         | 15683 |
| therubyracer | 14081 |
| pg           | 14029 |
| unicorn      | 13696 |
```

最受欢迎的宝石并不总是一致的。这可能是因为一些 gem(rake 和 json)是每次 Ruby 安装时安装的默认 gem。这将导致它们比任何其他 gem 都有更高的下载量，并且不会出现在 gem 文件中，因为它们被认为已经被下载了。

这种差异就是为什么使用两个或更多的数据源是一个好主意，如果你试图概括什么是最流行的。在选举季节[人们做预测](http://fivethirtyeight.com/)结合来自几次民意调查的信息，并使用其他数据得出他们的结论。我们可以通过结合 Rubygems.org 和 GitHub 的数据来确定最受欢迎的红宝石。

如果你想玩 GitHub 数据，你可以访问它[这里](https://cloudplatform.googleblog.com/2016/06/GitHub-on-BigQuery-analyze-all-the-open-source-code.html)。

07/15/16

*原载于 2016 年 7 月 15 日*[*www.thagomizer.com*](http://www.thagomizer.com/blog/2016/07/15/ruby-meets-bigquery-part-two.html)*。*