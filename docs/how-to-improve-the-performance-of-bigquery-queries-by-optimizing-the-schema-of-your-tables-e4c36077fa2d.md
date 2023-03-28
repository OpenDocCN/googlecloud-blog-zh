# 如何通过优化表的模式来提高 BigQuery 查询的性能

> 原文：<https://medium.com/google-cloud/how-to-improve-the-performance-of-bigquery-queries-by-optimizing-the-schema-of-your-tables-e4c36077fa2d?source=collection_archive---------0----------------------->

## 提高存储级 BigQuery 性能的三个技巧:嵌套字段、地理类型和聚类

在本文中，我采用了一个真实的表，并以无损的方式更改了它的模式，以提高该表上的查询性能。

![](img/54d65c80c0d9538ebbb7762e707743b3.png)

优化数据的存储方式，以获得更好的查询性能。安妮·斯普拉特在 [Unsplash](https://unsplash.com/search/photos/beehive?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) 上的照片

## 要优化的查询

为了说明表模式得到了改进，我们必须测量真实数据集上真实查询的性能。我将使用美国环境保护署(EPA)对可吸入颗粒物(PM10)的观测数据。 [EPA PM10 每小时数据集](https://console.cloud.google.com/bigquery?p=bigquery-public-data&d=epa_historical_air_quality&t=pm10_hourly_summary&page=table)作为 BigQuery 公共数据集程序的一部分可用。

这原本是一个扁平的表—基本上，每个小时的观察都有一行。数据集相对较小(1.4 GB，40m 行)，因此查询应该非常适合每月的免费配额(1 TB)。因为它是一张小桌子，所以改进不会像在大桌子上那样显著。

假设我们想知道 2017 年每个县有多少台 PM10 观测仪器。该查询是:

```
SELECT
 pm10.county_name,
 COUNT(DISTINCT pm10.site_num) AS num_instruments
FROM 
  `bigquery-public-data`.epa_historical_air_quality.pm10_hourly_summary as pm10
WHERE 
  EXTRACT(YEAR from pm10.date_local) = 2017 AND
  pm10.state_name = 'Ohio'
GROUP BY pm10.county_name
```

该查询耗时 2.4 秒，处理了 1.3 GB。

对于第二个查询，假设我们想找到俄亥俄州哥伦布市每年的最大 PM10 读数。城市多边形位于另一个公共数据集中，因此我们将连接它们:

```
SELECT
  MIN(EXTRACT(YEAR from pm10.date_local)) AS year
  , MAX(pm10.sample_measurement) AS PM10
FROM 
  `bigquery-public-data`.epa_historical_air_quality.pm10_hourly_summary as pm10
CROSS JOIN
  `bigquery-public-data`.utility_us.us_cities_area as city
WHERE
  pm10.state_name = 'Ohio' AND
  city.name = 'Columbus, OH' AND
  ST_Within( ST_GeogPoint(pm10.longitude, pm10.latitude), 
             city.city_geom )
GROUP BY EXTRACT(YEAR from pm10.date_local)
ORDER BY year ASC
```

这花费了大约 4 分钟，处理了 1.4 GB，并产生了哥伦布这些年的 PM10 读数。

这是我将用来演示优化的两个查询。

## 技巧 1:使用嵌套字段

EPA 每小时的数据在一个表中，表中的每一行都是每小时的观察值。这意味着现在有许多关于车站等的重复数据。让我们将同一天来自同一个传感器的所有观察值组合成一个数组(参见下面的 ARRAY_AGG ),并将它写入一个新表(首先创建一个名为 advdata 的新数据集):

```
CREATE OR REPLACE TABLE advdata.epa ASSELECT
  state_code
  , county_code
  , site_num
  , parameter_code
  , poc
  , MIN(latitude) as latitude
  , MIN(longitude) as longitude
  , MIN(datum) as datum
  , MIN(parameter_name) as parameter_name
  , date_local
  , **ARRAY_AGG**(STRUCT(time_local, date_gmt, sample_measurement, uncertainty, qualifier, date_of_last_change) ORDER BY time_local ASC) AS obs
  , STRUCT(MIN(units_of_measure) as units_of_measure
         , MIN(mdl) as mdl
         , MIN(method_type) as method_type
         , MIN(method_code) as method_code
         , MIN(method_name) as method_name) AS method
  , MIN(state_name) as state_name
  , MIN(county_name) as county_name
FROM `bigquery-public-data.epa_historical_air_quality.pm10_hourly_summary`
GROUP BY state_code, county_code, site_num, parameter_code, poc, date_local
```

新表的行数更少(170 万)，但仍然是 1.41 GB，因为我们没有丢失任何数据！不同之处在于，我们将观察到的值存储为一行中的数组。因此，行数减少到了原来的 1/24。

按县查询仪器现在是:

```
SELECT
 pm10.county_name,
 COUNT(DISTINCT pm10.site_num) AS num_instruments
FROM 
  advdata.epa as pm10
WHERE 
  EXTRACT(YEAR from pm10.date_local) = 2017 AND
  pm10.state_name = 'Ohio'
GROUP BY pm10.county_name
```

现在，查询耗时 0.7 秒(速度提高了 3 倍)，处理量仅为 56 MB(成本降低了 24 倍)。为什么不那么贵？因为少了 24 行(还记得我们将每小时的测量值聚集成一行)，所以表扫描需要处理的数据少了 24 行。查询速度更快，因为它需要处理的行数更少。

但是如果你真的需要处理每小时的数据呢？由于使用数组的转换没有损失，我们仍然可以查询哥伦布这些年来每小时最大的 PM10 观测值。该查询现在在 FROM 子句中需要一个 UNNEST，但在其他方面是相同的:

```
SELECT
  MIN(EXTRACT(YEAR from pm10.date_local)) AS year
  , MAX(**pm10obs.**sample_measurement) AS PM10
FROM 
  advdata.epa as pm10,
  **UNNEST(obs) as pm10obs**
CROSS JOIN
  `bigquery-public-data`.utility_us.us_cities_area as city
WHERE 
  city.name = 'Columbus, OH' AND
  ST_Within( ST_GeogPoint(pm10.longitude, pm10.latitude), 
             city.city_geom )
GROUP BY EXTRACT(YEAR from pm10.date_local)
ORDER BY year ASC
```

这个查询仍然需要 4 分钟，但是它只处理 537 MB。换句话说，将数据存储为嵌套字段(数组)使查询成本降低了 3 倍！这很奇怪。为什么读取的数据会下降？因为有些行(Columbus 之外的行)不需要读取数组数据。但是计算(max，extract year，ST_Within)是这个查询的大部分开销，并且执行的行数是相同的，所以查询速度没有变化。

## 技巧 2:地理类型

我们能通过更好地存储数据来改进计算吗？是啊！

与其每次都从经纬度构造地理点，不如将经纬度存储为地理类型。原因在于，使用 ST_GeogPoint()创建地理点实际上是一项开销较大的操作，需要查找包含该点的 S2 像元(如果您试图创建更复杂的形状，如多边形，则开销会更大):

```
CREATE OR REPLACE TABLE advdata.epageo ASSELECT 
  * except(latitude, longitude)
  , ST_GeogPoint(longitude, latitude) AS location
FROM advdata.epa
```

第一个查询是相同的，因为我们在查询中不使用纬度和经度。第二个查询现在可以避免创建 ST_GeogPoint:

```
SELECT
  MIN(EXTRACT(YEAR from pm10.date_local)) AS year
  , MAX(pm10obs.sample_measurement) AS PM10
FROM 
  advdata.**epageo** as pm10,
  UNNEST(obs) as pm10obs
CROSS JOIN
  `bigquery-public-data`.utility_us.us_cities_area as city
WHERE 
  city.name = 'Columbus, OH' AND
  ST_Within( **pm10.location**, city.city_geom )
GROUP BY EXTRACT(YEAR from pm10.date_local)
ORDER BY year ASC
```

它耗时 3.5 分钟，处理 576 MB，也就是说，在 12.5%的查询性能加速下，多了 6%的数据(一个点的 geography 类型使用的数据比两个 floats 多)。

## 技巧#3:集群

请注意，我们非常广泛地使用时间。如果我们要求 BigQuery 以这样一种方式存储它的表，即该字段的所有相等值都保存在相邻的行中，会怎么样呢？这样，如果我们的查询在某个时间点进行过滤，BigQuery 就不必进行全表扫描。相反，它只能读取表的一部分。

因为您一次只能创建 2000 个分区，所以我决定通过一个虚拟日期字段进行分区(这没问题，因为通过集群，我们强制查询使用分区):

```
CREATE OR REPLACE TABLE advdata.epaclustered 
PARTITION BY dummy_month
CLUSTER BY state_name, date_local
AS 
SELECT 
*, 
CAST(
CONCAT(CAST(EXTRACT(YEAR from date_local) AS STRING), "-", 
       CAST(EXTRACT(MONTH from date_local) AS STRING), "-01") AS DATE) AS dummy_month
FROM advdata.epageo
```

现在，让我们来看第一个查询:

```
SELECT
 pm10.county_name,
 COUNT(DISTINCT pm10.site_num) AS num_instruments
FROM 
  advdata.epaclustered as pm10
WHERE 
  pm10.state_name = 'Ohio'
GROUP BY pm10.county_name
```

0.8 秒，43MB！没有区别。

第二个呢？

```
SELECT
  MIN(EXTRACT(YEAR from pm10.date_local)) AS year
  , MAX(pm10obs.sample_measurement) AS PM10
FROM 
  advdata.epaclustered as pm10,
  UNNEST(obs) as pm10obs
CROSS JOIN
  `bigquery-public-data`.utility_us.us_cities_area as city
WHERE 
  pm10.state_name = 'Ohio' AND
  city.name = 'Columbus, OH' AND
  ST_Within( pm10.location, city.city_geom )
GROUP BY EXTRACT(YEAR from pm10.date_local)
ORDER BY year ASC
```

现在，查询只需 20 秒，处理 576.4 MB，速度提高了 10 倍。这是因为我们按月对表进行了聚类，并按月进行了筛选，这使得 BigQuery 能够更有效地组织数据。

尽情享受吧！