# 熊猫-gbq 版本 0.3.0

> 原文：<https://medium.com/google-cloud/pandas-gbq-version-0-3-0-afd3afa25add?source=collection_archive---------1----------------------->

我们刚刚发布了熊猫的 0.3.0 版本[-gbq](https://github.com/pydata/pandas-gbq/releases/tag/0.3.0)。

![](img/18b74724f6f79571ed4fad65dce1007a.png)

pandas-gbq 0.3.0 支持使用数组和结构列的查询

*   [PyPI 发布](https://pypi.org/project/pandas-gbq/0.3.0/)
*   [康达锻造发布](https://anaconda.org/conda-forge/pandas-gbq/files?version=0.3.0)

此版本包括一些主要的增强功能和错误修复。

*   当您的查询结果包含一个[结构类型列](https://cloud.google.com/bigquery/docs/reference/standard-sql/data-types#struct-type)时，库现在使用一个字典来表示相应 dataframe 列中的值。
*   同样，当查询结果包含一个[数组类型的列](https://cloud.google.com/bigquery/docs/reference/standard-sql/data-types#array-type)时，库现在使用一个列表来表示相应 dataframe 列中的值。
*   *to_gbq()* 函数从数据帧中创建一个 BigQuery 表，变得更加可靠。它现在使用一个[加载作业](https://cloud.google.com/bigquery/docs/loading-data-local)将行添加到一个表中，而不是流 API，这对于新的和最近重新创建的表有一些一致性问题。

为了实现这一点，pandas-gbq 现在使用 [Google Cloud client library 进行 BigQuery](https://googlecloudplatform.github.io/google-cloud-python/latest/bigquery/usage.html) 。如果您正在升级您的环境，也要注意[安装新的依赖关系](https://pandas-gbq.readthedocs.io/en/latest/install.html#dependencies)。

感谢所有对此版本做出贡献的人。