# Firestore/Datastore:在 Go 中解锁查询过滤器功能

> 原文：<https://medium.com/google-cloud/firestore-datastore-unlocked-the-query-filter-capabilities-in-go-8cc5bf0e9af2?source=collection_archive---------0----------------------->

![](img/1a8092b363cbc35f4476425fe55d91f5.png)

当您必须存储面向文档的数据时，您希望将它们存储在**文档数据库**中。Google 云平台提出了两种类型的文档 DB 引擎: [**Datastore**](https://cloud.google.com/datastore/docs/concepts/overview) 和 [**Firestore**](https://cloud.google.com/firestore/docs) 。这两种产品非常相似，但故事不同。Datastore 来自 App Engine，Firestore 来自 Firebase suite *(一部分文档仍在 Firebase 域下)*。

# 数据存储/Firestore 概述

产品有**无服务器**和**随流量自动缩放**。您有到达平台的 API，您可以创建具有**事务和批处理功能**的文档。

一个强大的特性是在文档创建、更新或删除时触发事件的能力。您可以在这个事件上插入一个云函数来执行额外的处理。*例如，用于净化博客帖子消息。*

另一个相似之处是，**对执行的操作数量**(读/写/删除)和存储的数据量(包括索引大小)负责。

# 查询限制

**两款产品在优势和定价上非常相似**，但也有各自的弱点。最主要的一个是**[**查询限制**](https://firebase.google.com/docs/firestore/query-data/queries#query_limitations) 。**

*   **不能用`!=`排除一个值。
    您必须创建两个范围条件:`>`和`<`以获得相同的结果**
*   **如果您使用`>`和/或`<`范围条件，它必须在同一个字段上。不可能有几个字段具有`>`或`<`条件**
*   **`IN`子句的等价物最多被限制为 10 个元素**
*   **当您在一个字段上筛选范围和在另一个字段上筛选相等时，您需要预先创建一个复合索引**
*   **不可能对嵌套字段进行筛选**

# **后处理解决方案**

**在我的 Go 项目中，我处理了这些问题，我有**个困难来满足产品所有者的期望**。**

> **消费者必须能够使用任何条件过滤任何字段**

**这就是为什么我**开发了** [**JsonFilter 库**](https://pkg.go.dev/github.com/guillaumeblaquiere/jsonFilter) 来 post 处理 Datastore/Firestore 结果**(但是，最终，也可以应用于任何 struct 数组)。****

**标准流程如下**

*   **获取过滤字符串并根据查询的目标结构“编译”它**
*   ***将数据查询到 Firestore/Datastore 中，并将结果映射到 struct 数组中(不由库执行)***
*   ****应用过滤器来消除**与过滤器不匹配的数组条目。**

**让我们更深入地了解这个库的用法、过滤功能以及如何定制它。**

## **[过滤器编译](https://pkg.go.dev/github.com/guillaumeblaquiere/jsonFilter?tab=doc#Filter.Init)**

**过滤器串被提交给库，用于针对结构数组的结构类型的**编译(通常是 Firestore/Datastore 查询结果)。****

```
type structExample struct {...}...filter := jsonFilter.Filter{}

if filterValue != "" {
   err := filter.Init(filterValue, structExample{})
   if err != nil {
      //*TODO error handling
*   }
}
```

**编译通过反射检查**包含在过滤器定义**中的字段是否存在于参数中的结构类型**中。以及**类型是否符合**；例如，您不能对字符串类型应用范围过滤器。*文档中描述了可能的错误。*****

***考虑复杂和嵌套类型(指针、对另一类型的引用、映射/数组/矩阵结构)***

## **可定制的过滤器格式**

**根据我的需要，我为过滤器定义了一种格式。这种格式**非常适合通过 HTTP GET URL 参数**提供的字符串(我的用例)**

```
key1=val1,val2:key2.subkey=val3
```

**其中:**

*   **`key1`是要过滤的 JSON 字段名。你可以使用组合过滤器来浏览你的 JSON 树，就像`key2.subkey`**
*   **`=`是操作员。`!=` `>` `<`也可用**
*   **`val1`、`val2`、`val3`是要比较的值**
*   **这些值由逗号分隔`,`**
*   **不同的字段由冒号分隔`:`**

**然而，这种固执己见的**格式可能无法与**的所有用例相匹配。您可能还想**限制嵌套字段的深度扫描**。为此，[您可以定制您的过滤器选项](https://pkg.go.dev/github.com/guillaumeblaquiere/jsonFilter?tab=doc#Filter.SetOptions)**

```
filter := jsonFilter.Filter{}

	o := &jsonFilter.Options{
		MaxDepth:             		4,
		EqualKeyValueSeparator:    	"=",
  		GreaterThanKeyValueSeparator: 	">",
		LowerThanKeyValueSeparator:   	"<",
		NotEqualKeyValueSeparator:    	"!=",
		ValueSeparator:       		",",
		KeysSeparator:        		":",
		ComposedKeySeparator: 		".",
	}

	filter.SetOptions(o)
```

## **[应用过滤器](https://pkg.go.dev/github.com/guillaumeblaquiere/jsonFilter?tab=doc#Filter.ApplyFilter)**

**您可以**在 struct** 的数组上应用过滤器(与编译/初始化步骤中提供的 struct 类型相同)。*通常，在应用过滤器之前，Firestore/Datastore 查询结果必须映射到 struct 数组中。***

**该动作扫描数组中的所有元素，并由**驱逐不匹配的元素。****

```
results := getDummyExamples()if filterValue != "" {
   ret, err := filter.ApplyFilter(results)
   if err != nil {
      //*TODO error handling
*   }
   results = ret.([]structExample)
}
```

****就这些。所有不想要的值都从结果中删除**，遵循以下规则:**

*   **过滤器的每个元素必须返回 OK 以保持条目值**
*   **等式`=`，相当于 sql 子句中的:至少有一个值必须匹配。**
*   **not equality `!=`，相当于 sql 子句中的 not:所有值不能匹配。**
*   **大于`>`和小于`<`:只能比较适用的一个数字。**

# **解决的问题和限制**

**该库的目标是提出 Firestore/Datastore 默认提供的更高级的过滤器。它允许您解除限制:**

*   **允许过滤任何类型和嵌套类型(映射、数组、数组/对象的映射、映射/对象的数组等)**
*   **允许在数据集上使用几个过滤器`IN`**
*   **在`IN`条件下使用超过 10 个元素**
*   **允许在数据集上使用多个过滤器`NOT IN`**
*   **允许用`>`和`<`操作符比较几个范围**
*   **不需要创建任何复合索引**

**由此，**所有的原生限制都被解决了！****

## **性能问题**

**因为库扫描所有 Firestore/Datastore 结果，所以性能可能是个问题。**其实，它*应该*不会。****

**事实上，过滤器**应该只应用在一个小的 struct** 数组上，并且过滤开销非常小。如果你阅读成千上万的文件，你将会付出很多代价！**

**此外，由于需要恢复大量文档和过滤持续时间，您的 API 响应时间将会更长。**

****库是为方便消费者**使用他们的过滤器而设计的，它有**不被用作核心数据库特性**。**

# **有什么愿望或想法吗？**

**我有更新图书馆的想法，但我没有实施，因为我不需要它们。举个例子，**

*   **没有像*这样的通配符来替换任何 JSON 字段名或其一部分。**
*   **过滤器值没有通配符，如*或正则表达式功能。**

***根据我的需要，我将实现这些进化或其他*。**

**如果你有特殊要求，我将很乐意讨论和改进这个库。不要犹豫**打开关于** [**Github 项目**](https://github.com/guillaumeblaquiere/jsonFilter) 的问题并为之做出贡献！**