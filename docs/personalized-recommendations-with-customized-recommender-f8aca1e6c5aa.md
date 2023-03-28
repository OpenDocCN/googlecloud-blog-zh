# 具有定制推荐器的个性化推荐

> 原文：<https://medium.com/google-cloud/personalized-recommendations-with-customized-recommender-f8aca1e6c5aa?source=collection_archive---------3----------------------->

![](img/aa6b5e28610ec1341c0666ce53e536b9.png)

任何云平台的优势之一是可扩展性。现在创建**几十个**账户、**几百个**项目、**几千个**资源……非常容易，过一段时间后，你会有**吨的东西**，很难管理、优化和策划。
在谷歌云上， [**主动协助**](https://cloud.google.com/recommender/docs) 通过提供建议，帮助你保持你的云环境**干净、安全、高效**。

不同的推荐引擎**非常有用**并提供**强大的洞察力**。然而，这种见解是基于通用规则的。
例如，在 **90 天的观察期**之后，您将会得到 **IAM 建议**(减少对某个账户授予的权限)。根据您的使用情况，可能是**太多**，或者**太少**。

> 但是现在，你可以定制推荐者的配置！

让我们试试我最喜欢的推荐者: [**IAM 推荐者**](https://cloud.google.com/policy-intelligence/docs/role-recommendations-overview)

# 获取当前配置

第一步是**提取当前配置**。为此，您必须拥有推荐者查看者角色(在我们的示例中是 *IAM 推荐者查看者*角色)。

然后，你必须在[列表](https://cloud.google.com/recommender/docs/recommenders)中选择**你的推荐人的名字**。
我的是`google.iam.policy.Recommender`

最后，使用 **gcloud CLI 获取配置**并可视化结果

```
gcloud beta recommender recommender-config describe \
 google.iam.policy.Recommender \
 --location=global \
 --project=<ProjectID>
```

典型的结果如下

```
etag: '"24512cd0d91389e6"'
name: projects/<project>/locations/global/recommenders/google.iam.policy.Recommender/config
recommenderGenerationConfig:
  params:
    minimum_observation_period: P90D
revisionId: DEFAULT
updateTime: '2022-08-08T01:18:29Z'
```

*你可以注意到* `*P90D*` *默认定义了 90 天的观察期*

# 使用 CLI 应用您自己的配置

要使用 CLI**更新您的配置**，您需要两件事情:

*   当前参数
*   `ETAG`。
    *`*etag*`*通常用于了解请求者读取的最新版本。如果提交的* `*etag*` *与当前的相同，则接受更新，否则拒绝更新。**

## *获取并更新当前参数*

*您可以使用 **describe 命令**和`JQ`提取当前参数，并**将结果保存在文件**中，此处`paramsConfig.json`为*(JSON 格式)**

```
*gcloud beta recommender recommender-config describe \
 google.iam.policy.Recommender \
 --location=global \
 --project=<ProjectID> \
 --format=json \
 | jq .recommenderGenerationConfig > paramsConfig.json*
```

*然后，**更新参数值**。例如，*`*P30D*`*默认为 30 天的可观测性而不是 90 天。***

```
**{
  "params": {
    "minimum_observation_period": "P30D"
  }
}**
```

## **`ETAG`值**

**接下来，`etag`值。同样，用 describe 和`JQ`，但是要把**的结果保存在变量**中，这里用`ETAG`。**

```
**export ETAG=$(gcloud beta recommender recommender-config describe \
 google.iam.policy.Recommender \
 --location=global \
 --project=<ProjectID> \
 --format=json \
 | jq .etag)**
```

## **执行更新**

**最后，将所有这些放在**一个最终命令**中。使用`paramsConfig.json`和`etag`值**

```
**gcloud beta recommender recommender-config update \
 google.iam.policy.Recommender \
 --location=global \
 --project=<ProjectID> \
 --config-file=paramsConfig.json --etag=${ETAG}**
```

***您必须拥有推荐者管理员角色(在我们的示例中是* `*IAM*` *推荐者管理员角色)。***

**该命令成功应用后，可以再次**执行一次描述**(第一部分)**来确认**设置了正确的值。**

# **使用 API 的最简单方法**

**如你所见，**开发者体验并不好**。提取一部分 API 响应，单独获取 etag，好无聊。**

> **对于最简单的更新，您可以使用[推荐 API](https://cloud.google.com/recommender/docs/reference/rest/v1beta1/projects.locations.recommenders/updateConfig) 。**

## **获取并更新当前参数**

**首先，**在 JSON 中原样获取当前配置**，并将结果保存在一个文件中，例如*`*recommender-iam.json*`***

***或者像以前一样使用 CLI。***

```
***gcloud beta recommender recommender-config describe \
 google.iam.policy.Recommender \
 --location=global \
 --project=<ProjectID> \
 --format=json \
> [recommender](http://twitter.com/recommender)-iam.json***
```

***或者直接使用 API***

```
***curl -H "x-goog-user-project: <ProjectID>" \
 -H "Authorization: Bearer $(gcloud auth print-access-token)" \
[https://recommender.googleapis.com/v1beta1/projects/<ProjectID>/locations/global/recommenders/google.iam.policy.Recommender/config](https://recommender.googleapis.com/v1beta1/projects/751286965207/locations/global/recommenders/google.iam.policy.Recommender/config) \
 > [recommender](http://twitter.com/recommender)-iam.json***
```

****请注意，您可以使用 CLI 获取要验证的访问令牌。
如果你用的是你的用户账号，那就不得不用* `*x-goog-user-project*` *头来提“消费项目”。
如果您使用服务帐户，您可以将其删除。****

**保存后，*更新内容*；*以*为例，将 `*P90D*` *改为* `*P30D*`**

## ***执行更新***

***有趣的部分来了。**保持提取的 JSON 不变**。没有要提取的`etag`或参数！***

```
***curl -H "x-goog-user-project: <ProjectID>" \
 -d [@recommender](http://twitter.com/recommender)-iam.json -X PATCH \
 -H "Authorization: Bearer $(gcloud auth print-access-token)" \
 -H "Content-Type: application/json" \
[https://recommender.googleapis.com/v1beta1/projects/<ProjectID>/locations/global/recommenders/google.iam.policy.Recommender/config](https://recommender.googleapis.com/v1beta1/projects/751286965207/locations/global/recommenders/google.iam.policy.Recommender/config)***
```

****我已经和工程团队分享了这种最简单的方法。希望 CLI 快点好起来！****

# **用你的规则搭建的平台**

**推荐者定制仅在开始才**并且所有的推荐者还不可定制。****

**此外，一些**强制组件缺少**，如 Terraform 模块能够直接使用 IaC 设置推荐器参数。**

**无论如何，您可以**开始思考和定义您的策略**以及您**希望如何被推荐**来优化您的云环境！**