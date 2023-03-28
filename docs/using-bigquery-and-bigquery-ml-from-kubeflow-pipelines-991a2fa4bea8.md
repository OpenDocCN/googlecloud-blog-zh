# 使用 Kubeflow 管道中的 BigQuery(和 BigQuery ML)

> 原文：<https://medium.com/google-cloud/using-bigquery-and-bigquery-ml-from-kubeflow-pipelines-991a2fa4bea8?source=collection_archive---------0----------------------->

## 使用 Python 函数来组合功能

这些天，当有人问我建立机器学习开发到操作化工作流的最佳方式时，我会告诉他们 [Kubeflow Pipelines](https://www.kubeflow.org/docs/pipelines/overview/pipelines-overview/) (KFP)。在谷歌云上，[云人工智能平台管道](https://cloud.google.com/ai-platform/pipelines/docs)为 KFP 提供了托管体验，这样你就不必在 Kubernetes 集群上瞎折腾了。

![](img/b32d7f4d457864022962ba944501de2d.png)

## 调用 BigQuery 的 Python 代码

我们首先编写一个函数，该函数使用 BigQuery Python 客户端运行 BigQuery 查询，该查询创建一个表并返回表或模型名称:

```
from typing import NamedTupledef run_bigquery_ddl(project_id: str, query_string: str, 
    location: str) -> NamedTuple(
    'DDLOutput', [('created_table', str), ('query', str)]):
    """
    Runs BigQuery query and returns a table/model name
    """
    print(query_string)

    from google.cloud import bigquery
    from google.api_core.future import polling
    from google.cloud import bigquery
    from google.cloud.bigquery import retry as bq_retry

    bqclient = bigquery.Client(project=project_id, location=location)
    **job = bqclient.query(query_string, retry=bq_retry.DEFAULT_RETRY)**
    job._retry = polling.DEFAULT_RETRY

    while job.running():
        from time import sleep
        sleep(0.1)
        print('Running ...')

    **tblname = job.ddl_target_table**
    tblname = '{}.{}'.format(tblname.dataset_id, tblname.table_id)
    print('{} created in {}'.format(tblname, job.ended - job.started))

    from collections import namedtuple
    result_tuple = namedtuple('DDLOutput', ['created_table', 'query'])
    **return result_tuple(tblname, query_string)**
```

## 使用容器函数创建容器

在 Kubeflow 管道中，每个步骤(或“操作”)都需要是一个容器。幸运的是，使用上面的 Python 函数并使其成为一个容器非常简单:

```
ddlop = comp.func_to_container_op(run_bigquery_ddl,
                    packages_to_install=['google-cloud-bigquery'])
```

注意，我指定了函数所依赖的 Python 包。还要仔细观察 Python 函数本身——任何非标准的包都被导入到函数定义中的*。*

## 使用容器

给定上面的容器(ddlop ),我们可以使用它来执行我们想要的任何表或模型创建查询。例如，下面是一个训练模型的查询，作为管道的一部分调用:

```
def train_classification_model(ddlop, project_id):
    query = """
        CREATE OR REPLACE MODEL mlpatterns.classify_trips
        TRANSFORM(
          trip_type,
          EXTRACT (HOUR FROM start_date) AS start_hour,
          EXTRACT (DAYOFWEEK FROM start_date) AS day_of_week,
          start_station_name,
          subscriber_type,
          ML.QUANTILE_BUCKETIZE(member_birth_year, 10) OVER() AS bucketized_age,
          member_gender
        )
        OPTIONS(model_type='logistic_reg', 
                auto_class_weights=True,
                input_label_cols=['trip_type']) ASSELECT
          start_date, start_station_name, subscriber_type, member_birth_year, member_gender,
          IF(duration_sec > 3600*4, 'Long', 'Typical') AS trip_type
        FROM `bigquery-public-data.san_francisco_bikeshare.bikeshare_trips`
    """
    print(query)
    return ddlop(project_id, query, 'US')
```

## 编写管道

我们可以将这样的查询串成一个 ML 管道:

```
[@dsl](http://twitter.com/dsl).pipeline(
    name='Cascade pipeline on SF bikeshare',
    description='Cascade pipeline on SF bikeshare'
)
def cascade_pipeline(
    project_id = PROJECT_ID
):
    ddlop = comp.func_to_container_op(run_bigquery_ddl, packages_to_install=['google-cloud-bigquery'])

    c1 = train_classification_model(ddlop, PROJECT_ID)
    c1_model_name = c1.outputs['created_table']

    c2a_input = create_training_data(ddlop, PROJECT_ID, c1_model_name, 'Typical')
    c2b_input = create_training_data(ddlop, PROJECT_ID, c1_model_name, 'Long')

    c3a_model = train_distance_model(ddlop, PROJECT_ID, c2a_input.outputs['created_table'], 'Typical')
    c3b_model = train_distance_model(ddlop, PROJECT_ID, c2b_input.outputs['created_table'], 'Long')

    evalop = comp.func_to_container_op(evaluate, packages_to_install=['google-cloud-bigquery', 'pandas'])
    error = evalop(PROJECT_ID, c1_model_name, c3a_model.outputs['created_table'], c3b_model.outputs['created_table'])
    print(error.output)
```

现在，您可以创建这个管道的 zip 文件，并将其提交给 ML Pipelines 集群，以调用新的实验运行。

## 试试吧！

本文的[完整代码在 GitHub 上。要试用笔记本电脑:](https://github.com/GoogleCloudPlatform/ml-design-patterns/blob/master/03_problem_representation/cascade.ipynb)

*   按照[设置 AI 平台管道](https://cloud.google.com/ai-platform/pipelines/docs/setting-up)操作指南创建 AI 平台管道实例。创建 GKE 集群时，确保启用对[https://www.googleapis.com/auth/cloud-platform](https://www.googleapis.com/auth/cloud-platform)的访问。
*   在 GCP 控制台的 AI 平台/笔记本中创建一个笔记本实例(任何版本)。
*   克隆我的笔记本(上图)
*   更改第一个单元格，以反映您的 KFP 集群的主机名。

尽情享受吧！