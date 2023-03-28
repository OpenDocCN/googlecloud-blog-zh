# 谷歌云数据目录实践指南:Python 的模板和标签

> 原文：<https://medium.com/google-cloud/data-catalog-hands-on-guide-templates-tags-with-python-c45eb93372ef?source=collection_archive---------0----------------------->

这份快速入门指南是一个系列的一部分，它为**数据目录**带来了一种实践方法，这是[最近宣布的**谷歌云的数据分析**服务家族的](https://cloud.google.com/blog/products/data-analytics/google-cloud-data-catalog-now-available-in-public-beta)成员。

> [Data Catalog 是一种全面管理和可扩展的元数据管理服务，使组织能够快速发现、了解和管理他们在 Google Cloud 中的数据](https://cloud.google.com/data-catalog/)。

![](img/9331646e33fee0b7ead4e4904dc156ae.png)

以下内容显示了如何将数据目录的*标记*功能用于 Python GRPC 客户端库。这个故事是[数据目录动手指南:搜索，用 Python](/google-cloud/data-catalog-hands-on-guide-search-get-lookup-with-python-82d99bfb4056) 获得&查找的续篇，所以我建议在开始这个之前先读上一篇。如果你在实践之前需要更多的概念背景，[请看看我写的描述我关于这些特性的心理模型的文章](/google-cloud/data-catalog-hands-on-guide-a-mental-model-dae7f6dd49e)。

# 环境设置

运行示例所需的环境与[数据目录实践指南:使用 Python](/google-cloud/data-catalog-hands-on-guide-search-get-lookup-with-python-82d99bfb4056) 搜索、获取&查找中描述的环境相同。详情请参考**环境设置**部分。

# 使用 Python 的模板和标签

正如我们已经看到的，*搜索目录*、*查找条目*和*获取条目*是在谷歌云项目中发现数据的有用工具。但是一旦我们知道并理解了这样的数据，我们还能做什么呢？我们如何利用数据目录来更好地管理它？

数据目录允许用户和自动化流程在 GCP 项目中标记数据。为了理解它是如何工作的，让我们探索一下*模板*和*标签*的特性。

## 模板

数据目录创建的所有*标签*都是基于模板的。`TagTemplate`是数据目录本地实体，表示元数据模式，但不是由`Entry`处理的同类元数据:标记模板表示定制/用户定义的元数据。或者，换句话说:`Entry`处理技术元数据，而`TagTemplate`处理业务元数据。

让我们描述一个`TagTemplate`来对我们使用*搜索*和*查找*找到的表进行分类。它暂时只有一个字段:**有 PII** ，一个**布尔**。根据[数据目录的标签模板规范](https://cloud.google.com/data-catalog/docs/reference/rest/v1beta1/projects.locations.tagTemplates#TagTemplate)，JSON 表示如下所示:

```
{
  "name": "..."
  "displayName": "..."
  "fields": {
    "has_pii": {
      "displayName": "Has PII"
      "type": {
        "primitiveType": BOOL
      }
    }
  }
}
```

很简单，不是吗？现在，看看使用 Python 客户端库创建它的代码:

```
**location** = f'projects/{**<project-id>**}/locations/us-central1'tag_template = datacatalog.TagTemplate()
tag_template.**display_name** = 'A Tag Template to be used in the hands-on guide'field = datacatalog.TagTemplateField()
field.**display_name** = 'Has PII'
field.**type_.primitive_type** = datacatalog.FieldType.PrimitiveType.**BOOL**
tag_template.**fields['has_pii']** = fielddatacatalog_client.create_tag_template(
    parent=location, 
    **tag_template_id**='quickstart_classification_template',
    tag_template=tag_template)
```

有什么问题吗？**代码的目的是不言自明的:)我们可以根据需要多次使用类似的代码来创建一个由几个字段组成的模板。**

> 为了获得预期的结果，服务帐户至少需要**数据目录标记模板创建者** IAM 角色。

数据目录的 API 中有一些方法允许我们修改一个`Tag Template`——即`create_tag_template_field`、`delete_tag_template_field`、`rename_tag_template_field`和`update_tag_template_field`。在我们探索 API 时，让我们看看`create_tag_template_field`是如何使用它向模板添加第二个字段的: **PII 类型**，一个带有值**电子邮件**和**社会安全号**的**枚举**。

```
field = datacatalog.TagTemplateField()**>>>** email_value = datacatalog.FieldType.EnumType.EnumValue()
email_value.display_name = 'EMAIL'
field.type_.**enum_type.allowed_values.append(email_value)**ssn_value = datacatalog.FieldType.EnumType.EnumValue()
ssn_value.display_name = 'SOCIAL SECURITY NUMBER'
field.type_.**enum_type.allowed_values.append(ssn_value)
<<<**datacatalog_client.create_tag_template_field(
    parent=tag_template_name, **tag_template_field_id**='pii_type', tag_template_field=field)
```

请注意>>>和<<< marks, where we set the enum values through the 【 [*之间的行重复消息字段*](https://developers.google.com/protocol-buffers/docs/reference/python-generated#repeated-message-fields) 。运行完这段代码后，我们得到了最后的`TagTemplate`:

```
name: "projects/**<project-id>**/locations/us-central1/tagTemplates/quickstart_classification_template"
display_name: "A Tag Template to be used in the hands-on guide"
fields {
  key: "has_pii"
  value {
    display_name: "Has PII"
    type {
      primitive_type: BOOL
    }
  }
}
fields {
  key: "pii_type"
  value {
    display_name: "PII Type"
    type {
      enum_type {
        allowed_values {
          display_name: "EMAIL"
        }
        allowed_values {
          display_name: "SOCIAL SECURITY NUMBER"
        }
      }
    }
  }
}
```

既然我们已经完成了模板，接下来让我们看看如何使用它来创建*标签*。

## 标签

`Tag`是另一个数据目录原生实体。它允许用户/服务帐户基于一个`TagTemplate`为给定的`Entry`附加业务元数据。

在[数据目录实践指南:使用 Python](/google-cloud/data-catalog-hands-on-guide-search-get-lookup-with-python-82d99bfb4056) 搜索、获取&查找中，我们看到了如何获取`table_1`和`table_2`的目录条目。我们在上一节中创建了一个模板，因此创建一个`Tag`所需的所有信息都是可用的。

看看下一段代码:

```
**(1)**
tag_1 = datacatalog.Tag()
tag_1.template = tag_template.namehas_pii_field = datacatalog.TagField()
has_pii_field.bool_value = **False**
tag_1.fields[**'has_pii'**] = has_pii_fielddatacatalog_client.create_tag(parent=**<table-1-entry>**.name, tag=tag_1)print(tag_1.name)**(2)**
tag_2 = datacatalog.Tag()
tag_2.template = tag_template.namehas_pii_field = datacatalog.TagField()
has_pii_field.bool_value = **True**
tag_2.fields[**'has_pii'**] = has_pii_fieldpii_type_field = datacatalog.TagField()
pii_type_field.enum_value.display_name = **'EMAIL'**
tag_2.fields[**'pii_type'**] = pii_type_fielddatacatalog_client.create_tag(parent=**<table-2-entry>**.name, tag=tag_2)print(tag_2.name)
```

它展示了如何使用相同的模板将具有不同值的标记附加到表中。简而言之，现在数据目录用户将知道`table_1`没有 PII 信息，而`table_2`存储电子邮件。

通过设置`tag.column = column_name`，可以将`Tag`附加到特定的表格列，而不是表格本身。

> 要成功附加标签，服务帐户至少需要**数据目录标签模板用户** IAM 角色。根据条目引用的数据资产类型，将需要额外的权限/自定义角色。例如，用于 bigquery 数据集的**BigQuery . Datasets . update tag**，用于 big query 表的**big query . Tables . update tag**，以及用于发布/订阅主题的**Pub Sub . Topics . update tag**。

这个片段的预期输出(名称)如下所示。请注意，标签是属于条目的资源，而不是属于模板的资源，标签的 id 是系统生成的。

```
projects/**<project-id>**/locations/US/entryGroups/@bigquery/entries/**<table-1-entry-id>**/tags/**<tag-1-id>**projects/**<project-id>**/locations/US/entryGroups/@bigquery/entries/**<table-2-entry-id>**/tags/**<tag-2-id>**
```

相同的方法用于将标签附加到 BigQuery 数据集或发布/订阅主题。现在不要耍花招。

# 重新访问搜索目录

既然我们标记了目录条目，这些信息如何用于将来的搜索？标签给*搜索目录*带来什么好处吗？是的，当然！`tag`搜索限定符允许我们寻找带有给定模板或值标签的资产，随着越来越多的分类工作的完成，这种搜索能力可以用来轻松地管理/审计数据。

例如:

```
datacatalog_client.search_catalog(scope=scope, query=**'tag:quickstart_classification_template'**)
```

将返回`table_1`和`table_2`的搜索结果，而

```
datacatalog_client.search_catalog(scope=scope, query=**'tag:quickstart_classification_template.has_pii=True'**)
```

将只为`table_2`返回一个结果。

# 搞定了。

本系列通过端到端的案例研究，从数据发现到用户定义的业务元数据管理，初步概述了 Google Cloud Data Catalog，并提供了一个心理模型作为背景。它们构成了使数据目录成为任何规模的公司中数据治理支持的理想工具的基本功能集。

还讲述了 Python 客户端库的基本用法。这些示例可以很容易地移植到 Java、NodeJS 或其他语言中([查看官方文档了解可用性](https://cloud.google.com/data-catalog/docs/reference/libraries))。此外，考虑使用[云控制台](https://console.cloud.google.com)中的数据目录，在开始时，你可能会获得有用的见解。

就这些了，伙计们！

> 用于生成上述代码样本的源文件可以在 GitHub 上找到:【https://github.com/ricardolsmendes/gcp-datacatalog-python。

# 参考

*   **数据目录入门指南**:[https://cloud . Google . com/Data-Catalog/docs/quick starts/quick start-search-tag](https://cloud.google.com/data-catalog/docs/quickstarts/quickstart-search-tag)
*   **谷歌云数据目录 API (Python)客户端**:[https://Google APIs . github . io/Google-Cloud-Python/latest/Data Catalog/gapic/v1 beta 1/API . html](https://googleapis.github.io/google-cloud-python/latest/datacatalog/gapic/v1beta1/api.html)
*   **Google 云数据目录 API 客户端(Python)的类型**:[https://Google APIs . github . io/Google-Cloud-Python/latest/Data Catalog/gapic/v1 beta 1/Types . html](https://googleapis.github.io/google-cloud-python/latest/datacatalog/gapic/v1beta1/types.html)
*   **Python 生成的代码|协议缓冲区|谷歌开发者**:[https://Developers . Google . com/Protocol-Buffers/docs/reference/Python 生成的](https://developers.google.com/protocol-buffers/docs/reference/python-generated)

# 变更日志

*   2020–10–03:更新了代码片段，以强制符合客户端库的 2.0.0 版。
*   2021–02–15:更新了代码片段，以强制遵守客户端库的[版本 3.0.0。](https://github.com/googleapis/python-datacatalog/blob/master/UPGRADING.md#300-migration-guide)