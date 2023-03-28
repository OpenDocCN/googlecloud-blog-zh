# "医学笔记——一种未被充分利用的资源."

> 原文：<https://medium.com/google-cloud/medical-notes-an-underutilized-resource-73c9e4b1a140?source=collection_archive---------1----------------------->

![](img/a79ee9969093dfb0623bab42d2677130.png)

## 充分利用 Google Healthcare 自然语言处理 API 从非结构化医学文本中获取见解。

答 *根据* [*健康医疗*](https://hitinfrastructure.com/news/unstructured-healthcare-data-needs-advanced-machine-learning-tools#:~:text=About%2080%20percent%20of%20healthcare,of%20Things%20(IoT)%20devices.) *杂志的报道，“80%的医疗保健数据是非结构化的。组织不仅需要工具来回顾他们已经存储的遗留数据，而且还必须处理每天产生的越来越多的数据。随着互联医疗和物联网(IoT)设备的增加，组织正在以惊人的速度收集非结构化数据。*”

从包裹递送到医疗保健以及中间的每个行业，收集越来越多数据的趋势是显而易见的。思想领袖认识到，在设备级别收集实时数据能够优化流程、获得早期警告、检测功能差距并实现一切自动化。总的来说，相同的数据可以用于更大的系统范围的分析，以确定模式，并且在医疗保健的情况下，改善人口健康结果。

尽管潜力巨大，但处理非结构化医疗数据(如护理人员记录)一直是一项挑战。一些医学术语的使用缺乏标准化意味着临床医生可能使用不同的术语来指代相同的药物或病症。即使潜在的语言和术语是相同的，全球各地独特的语音模式也使情况变得更加复杂。

到目前为止，医疗保健组织中的数据科学家如果希望将他们的医疗记录作为结构化数据用于数据科学，就必须构建完整的语言模型，对这些模型进行详尽的测试，然后——假设可以创建成功的模型——大规模部署该模型。达到这一点的过程是艰巨的，最终模型必须随着时间的推移不断迭代以保持最新。

Google Healthcare 自然语言处理(NLP) API 提供了一个 REST API，您可以将它用于自己的医学文本。它返回一个有组织的结构化数据集，带有映射到所用术语的标准化名称。这使得数据科学家可以使用这些标准化的名称来获得对疾病、药物等的见解，而无需自己创建这些映射。该过程减少了与这些类型的任务相关的潜在人为错误。医疗保健 NLP API 提供了如下详细信息:

*   主题(“谁或什么”)—该文本是关于患者、家庭成员还是关于家族病史？
*   时间相关性(“何时”)—这是当前状况，还是过去经历过的状况，或者可能是未来的诊断？
*   实体(“什么”)—文中提到的医疗对象。
*   链接实体(“什么”扩展)-映射到标准化名称的实体。
*   关系——直观描述实体之间关系的关系图。
*   可能性—给定语言的细微差别，算法对于给定语句的准确性有多大把握

*参考链接:*

[*使用医疗保健自然语言 API*](https://cloud.google.com/healthcare/docs/how-tos/nlp)

更多展示在下面的例子。

为了演示，我们将从包含医疗转录的 [Kaggle 数据集](https://www.kaggle.com/tboyle10/medicaltranscriptions)开始。然后，我们将构建一个 Apache Beam 管道，该管道包含医疗保健 NLP API，然后显示给定文本示例返回的结果。

数据管道建立在谷歌云平台上的 Dataflow 笔记本上，并基于 Github 上的[这个例子](https://github.com/rasalt/healthcarenlp/blob/main/nlp_public.ipynb)。虽然该示例显示了批处理，但是可以使用 [PubSubIO 连接器](https://beam.apache.org/releases/javadoc/2.26.0/org/apache/beam/sdk/io/gcp/pubsub/PubsubIO.html)来调整流记录的管道。

![](img/cfaafeb8a7c1a8553f434fa7011f41e5.png)![](img/25550e6b64a8ee9b425730a8ff816e1f.png)

他的管道将医疗保健 NLP API 结果存储在 BigQuery 中，这使得任何具有基本 SQL 技能的人都可以查询这些数据以获得洞察力。下面的图表只是从所能收集到的信息中触及了皮毛。此外，阅读和总结这一海量数据的管理负担已经消除，并在几分钟内得到处理。

我们现在可以评估给定程序的过去、即将发生和未来可能发生的事件的数量。例如，在获取临床记录的当天，实施了 297 次麻醉，我们可以看到还有 11 个程序被安排。这些数据可用于改进规划和优化资源利用。

![](img/e5b262a97bf95679a9ced2e3deb45846.png)

程序数量的近期/过去/未来总结

L 让我们来看看医学文本的实际样本和提供的结果:

主观:这位 23 岁的白人女性主诉过敏。她住在西雅图时曾经过敏，但她认为这里的情况更糟。过去，她尝试过 Claritin 和 Zyrtec。这两种方法都在短时间内奏效，但随后似乎失去了效力。她也用过 Allegra。她去年夏天用过，两周前又开始用了。它似乎工作得不是很好。她使用过非处方喷雾剂，但没有处方鼻腔喷雾剂。她确实患有哮喘，但不需要每天服药，也不认为哮喘会发作。她目前唯一的药物是 Ortho Tri-Cyclen 和 Allegra。过敏:，她没有已知的药物过敏。目标:生命体征:体重 130 磅，血压 124/78。HEENT:她的喉咙有轻度红斑，没有渗出物。鼻粘膜出现红斑和肿胀。只看到清晰的排水。TMs 很清楚。，颈部:柔软无腺病。，肺部:畅通。，评估:，过敏性鼻炎。，计划:，1。她将再次尝试 Zyrtec 而不是 Allegra。另一种选择是使用氯雷他定。她认为她没有处方保险，所以可能会更便宜。,2.在三周内，在每个鼻孔中喷两次。处方也写好了。

```
{"entityMentions": [ *← This array holds information about the entities identified in the medical transcription*
  {
    "mentionId": "1", ← **This is a unique id and you will use in in     “relationships” array** "type": "PROBLEM",
    "text": {
        "content": "allergies",
        "beginOffset": 71
     },
    "linkedEntities": [
       { 
          "entityId": "UMLS/1527304" ← **Look for this entity in the “Entities array”** },
       {
          "entityId": "UMLS/20517" ← **Look for this entity in the “Entities array”** }
    ],
    "temporalAssessment": {
      "value": "CURRENT", ←  **Is this current/past ?** "confidence": 0.9958259463310242
    }, "certaintyAssessment": { ← **Likelihood of the “PROBLEM”** "value": "LIKELY",
      "confidence": 0.9998759031295776
    },
    "subject": {
      "value": "PATIENT", ← The subject of the entity could be a patient or family member
      "confidence": 0.999998927116394
     },
    "confidence": 0.9866036176681519
  },… 
 ],
   "relationships": [ ← This maps relationships between the various "entityMentions"
     {
     "subjectId": "9", ← Looking up mentionId = 9 is PROBLEM:asthma
     "objectId": "11", ← Looking up mentionId = 11 is MEDICINE:
     "confidence": 0.7835226058959961
     }
...
 ]
}
```

现在您已经有了关于管道如何识别实体和关系的数据，下一步将是共享管道。

## 摘要

在这篇博客中，我们讨论了非结构化医疗数据的问题，以及如何使用谷歌云平台——特别是数据流、大查询和医疗保健 NLP API——来分析、评估并从中获得洞察力。