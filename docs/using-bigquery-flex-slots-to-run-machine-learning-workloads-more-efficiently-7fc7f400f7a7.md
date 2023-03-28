# 使用 BigQuery Flex 插槽更高效地运行机器学习工作负载

> 原文：<https://medium.com/google-cloud/using-bigquery-flex-slots-to-run-machine-learning-workloads-more-efficiently-7fc7f400f7a7?source=collection_archive---------0----------------------->

## 创建弹性插槽，运行查询，然后删除弹性插槽

根据您更看重什么，BigQuery 大致有两种不同的定价方案:可预测的成本或效率:

1.  **统一价格是可预测的**。你购买一定数量的插槽，并支付相同的金额，无论你提交多少查询。这对于需要可预测成本的企业来说非常好。缺点是，如果您购买了 1000 个插槽，并且有一个通常使用 500 个插槽，有时需要 1500 个插槽的高峰工作负载，当您只使用 500 个插槽时，插槽会被闲置，当拥有 1500 个插槽会更好时，查询会运行得更慢。
2.  **按需定价在价格和性能方面都很高效**。您需要为查询处理的数据量付费。这是最初的云承诺—按需付费。它非常适合尖峰工作负载，因为您不必为未使用的槽付费，并且您的查询使用了运行时所有可用的计算能力。然而，即使你可以设置成本控制，每天的价格限制等。为了限制你的暴露，许多企业不喜欢这样做，因为(a)很难为每月变化的事情做预算，以及(b)成本控制和每日限额增加了摩擦。

总的来说，我们发现较大的企业更喜欢统一费率，而较小的数字原生用户更喜欢按需定价。

![](img/2fef817e1683f41b2c5d55992d77c34d.png)

即使您支付统一费率，弹性插槽也能让您获得有效的定价。图片由 [PublicDomainPictures](https://pixabay.com/users/PublicDomainPictures-14/?utm_source=link-attribution&utm_medium=referral&utm_campaign=image&utm_content=2086) 提供。

## 灵活性

假设您可以采用统一费率定价，因为您想要可预测的成本，但是您有一个峰值工作负载。此外，您确切地知道*何时*您需要额外的容量。

假设您每天训练一次推荐模型，这时您需要 1500 个位置。其余时间，你只需要 500 个槽。购买一个 500 槽的统一费率计划，但在每天进行推荐模型培训的一个小时内额外购买 1000 槽，这不是很好吗？这样，您的常规工作负载不会受到影响，并且建议模型有 1000 个插槽，因此它不会运行得更慢。

这被称为[“柔性插槽”](https://cloud.google.com/blog/products/data-analytics/introducing-bigquery-flex-slots)。您可以购买短至 60 秒的灵活插槽。它允许您在统一价格的基础上增加灵活性。

## 即使采用按需定价，何时也要使用弹性时段

使用按需定价，您将为查询处理的数据量付费。如果您只是做一些基本的选择、WHERE、JOIN 和 GROUP BY，那么按需和统一费率的结果是一样的。对于计算量大的查询来说，按需定价很划算——如果您的查询涉及 g is、正则表达式解析、JSON 提取、ORDER BY 等。，只为处理的字节付费是很划算的。所以，总的来说，按需定价是非常划算的。

机器学习查询是个例外。机器学习模型训练需要对数据进行多次迭代，因此在按需情况下，您将根据 ML 模型可能需要进行的迭代次数收取费用。这意味着，如果[你知道](/google-cloud/bigquery-ml-gets-faster-by-computing-a-closed-form-solution-sometimes-1baa5a838eb6)你的 ML 模型[不会](https://towardsdatascience.com/k-means-clustering-in-bigquery-now-does-better-initialization-3d7e7567bad3)花费最大的时间，你最好使用统一费率预订或弹性时段——如果你为你实际使用的计算付费，BigQuery 中的 ML 会更便宜。

特别是，当数据和迭代收费时，推荐模型太昂贵了——常见的矩阵分解模型有数百万用户和数万种产品。[矩阵分解是一种 O(N)算法](http://rakaposhi.eas.asu.edu/s01-cse494-mailarchive/msg00028.html)。然而实际上，收敛更快，因为你的矩阵可能很稀疏。出于这个原因，如果您需要，BigQuery 拒绝进行矩阵分解，并告诉您购买 flex slots，以便您可以支付实际使用的计算费用。

## 在 Flex 插槽中运行查询

当然，您可以使用 BigQuery 控制台来创建 flex 插槽

![](img/c1277b789d880f6ad7c7061ed5230ce2.png)

但是您可以自动化这个流程——购买一些 flex slots，设置一个预订，运行 ML 模型训练查询，然后删除全部内容。这里有一个自动化整个过程的脚本。

假设我们想要对美国的 500 个 flex 插槽运行查询:

```
LOCATION=US
SLOTS=500
run_query() {
  cat ../bqml_recommendations/train.sql | bq query --sync -nouse_legacy_sql
}
```

首先购买具有灵活插槽的插槽容量预留:

```
bq mk --project_id=${PROJECT}  --location=${LOCATION} \
      --capacity_commitment --slots=${SLOTS} --plan=FLEX
```

然后，创建一个使用这些插槽的预留:

```
bq mk --reservation --project_id=${PROJECT} --slots=${SLOTS} \
      --location=${LOCATION} ${RESERVATION} || cleanup
```

如果创建预留失败，请确保清理槽(稍后将详细介绍)。

然后，允许我们的项目使用此保留:

```
bq mk --reservation_assignment \
      --reservation_id=${PROJECT}:${LOCATION}.${RESERVATION} \
      --job_type=QUERY --assignee_type=PROJECT \
      --assignee_id=${PROJECT}
```

上述三个步骤相当于您在 web 控制台上创建插槽、预订和分配的三个步骤:

![](img/631adc55d08fee4173607fef56651ca3.png)

现在，只需运行查询:

```
run_query
```

查询完成后，清理分配、预留和插槽提交(与创建顺序相反)

```
bq rm --reservation_assignment --project_id=${PROJECT} \
      --location=${LOCATION} ${ASSIGNMENT} || truebq rm --reservation --project_id=${PROJECT} \
      --location=${LOCATION}  ${RESERVATION} || trueuntil bq rm --location=${LOCATION}  --capacity_commitment ${CAPACITY}
  do
    echo "will try after 30 seconds to delete slots ${CAPACITY}"
    sleep 30
  done
  CAPACITY=""
```

如果脚本不能删除预留、分配或时段，它不会退出。它将继续运行。请更改此行为以提醒您。

这不是谷歌的官方作品。正如脚本的版权所言，该软件是在“原样”的基础上发布的，没有任何形式的保证或条件，无论是明示的还是暗示的。

## 后续步骤

1.  阅读更多关于[弹性插槽](https://cloud.google.com/blog/products/data-analytics/introducing-bigquery-flex-slots)的信息。
2.  查看我在 GitHub 上的[脚本](https://github.com/GoogleCloudPlatform/bigquery-oreilly-book/blob/master/blogs/flex_slots/run_query_on_flex_slots.sh)，它包含了一个关于创建和删除 flex slots 预约的查询。如果你对脚本有改进的建议，请向 GitHub 发出请求。
3.  阅读我的同事邓梓峰的文章，他展示了 Flex Slots 如何降低大型查询的成本。
4.  ML 模型的运行速度比数据的大小快得多的情况的例子表明:[具有< 1000 个特性的线性回归](/google-cloud/bigquery-ml-gets-faster-by-computing-a-closed-form-solution-sometimes-1baa5a838eb6)、 [kmeans++](https://towardsdatascience.com/k-means-clustering-in-bigquery-now-does-better-initialization-3d7e7567bad3) 、早期停止以及更多不断增加的优化。