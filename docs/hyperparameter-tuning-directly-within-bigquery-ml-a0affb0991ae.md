# 超参数调整 BigQuery ML

> 原文：<https://medium.com/google-cloud/hyperparameter-tuning-directly-within-bigquery-ml-a0affb0991ae?source=collection_archive---------1----------------------->

## BigQuery ML 可以使用顶点人工智能来调整通用模型参数

## 什么是 BigQuery ML？

BigQuery ML 允许您根据 BigQuery 中的数据快速训练 ML 模型。例如，假设您想要训练线性 ML 模型来预测自行车租赁的持续时间，您可以使用:

```
CREATE OR REPLACE MODEL ch09eu.bicycle_model_linear
OPTIONS(
 model_type='linear_reg', input_label_cols=['duration']
)
AS
SELECT 
  start_station_name,
  CAST(EXTRACT(DAYOFWEEK FROM start_date) AS STRING) AS dayofweek,
  CAST(EXTRACT HOUR FROM start_date) AS STRING) AS hourofday,
  duration
FROM `bigquery-public-data.london_bicycles.cycle_hire`
```

BigQuery ML 还支持在推理过程中自动重复的特征工程。稍微复杂一点的深度神经网络模型可能是:

```
CREATE OR REPLACE MODEL ch09eu.bicycle_model_dnn
**TRANSFORM(
   start_station_name,
   CAST(EXTRACT(DAYOFWEEK FROM start_date) AS STRING) AS dayofweek,
   ML.BUCKETIZE(EXTRACT(HOUR FROM start_date), [0, 6, 12, 18, 24]) AS hourofday,
   duration
)**
OPTIONS(
 **model_type='dnn_regressor'**, input_label_cols=['duration'], 
 hidden_units=[32, 8],
 learn_rate=0.1,
 dropout=0.25,
)
AS
SELECT 
  start_station_name, start_date, duration
FROM `bigquery-public-data.london_bicycles.cycle_hire`
```

## 超参数调谐

但是如果你想尝试不同的学习速度呢？这称为超参数调整，您可以指定模型参数的范围或候选值:

```
CREATE OR REPLACE MODEL ch09eu.bicycle_model_dnn_hparam
TRANSFORM(
   start_station_name,
   CAST(EXTRACT(DAYOFWEEK FROM start_date) AS STRING) AS dayofweek,
   ML.BUCKETIZE(EXTRACT(HOUR FROM start_date), [0, 6, 12, 18, 24]) AS hourofday,
   duration
)
OPTIONS(
 model_type='dnn_regressor', input_label_cols=['duration'], 
 **num_trials=10,**
 **hidden_units=hparam_candidates([struct([32,8])**, struct([32]), struct([64,32,4])]),
 **learn_rate=hparam_range(0, 0.5),
 dropout=hparam_candidates([0, 0.1, 0.25, 0.4])**
)
AS
SELECT 
  start_station_name, start_date, duration
FROM `bigquery-public-data.london_bicycles.cycle_hire`
```

BigQuery ML 将使用 Vertex AI 的超参数调优服务在您指定的预算(这里是 10 次试验)内选择最佳参数。

请注意，每种型号类型[只允许对少数参数](https://cloud.google.com/bigquery-ml/docs/reference/standard-sql/bigqueryml-hyperparameter-tuning#hyperparameters_and_objectives)进行调整。例如，对于 DNN 模型，您可以调整 batch_size、dropout、learn_rate、optimizer 等。

如果您想要更改存储桶间隔，该怎么办？或者试试有没有个性特色？在这种情况下，你应该[遵循本文](https://towardsdatascience.com/how-to-do-hyperparameter-tuning-of-a-bigquery-ml-model-29ba273a6563)中的方法，构建一个 Python 支架。这样的 Python 支架会给你很大的灵活性。

但是，对于调整基本模型参数，请使用内置的顶点 AI 集成。

## 调用最佳模型

培训完成后，您可以查看每个试验的评估指标:

```
SELECT * FROM ML.EVALUATE(MODEL ch09eu.bicycle_model_dnn_hparam) 
ORDER BY mean_absolute_error ASC
```

并调用最佳模型，使用:

```
SELECT * FROM ML.PREDICT(MODEL ch09eu.bicycle_model_dnn_hparam,
 (
   SELECT 
     CURRENT_TIMESTAMP() AS start_date, 
     'Waterloo Station 1, Waterloo' AS start_station_name
 )
)
```