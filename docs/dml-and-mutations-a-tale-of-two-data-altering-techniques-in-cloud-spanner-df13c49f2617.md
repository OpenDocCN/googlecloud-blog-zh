# DML 和突变——云扳手中两种数据更改技术的故事

> 原文：<https://medium.com/google-cloud/dml-and-mutations-a-tale-of-two-data-altering-techniques-in-cloud-spanner-df13c49f2617?source=collection_archive---------0----------------------->

数据操作语言(DML)和突变是 Cloud Spanner 中的两个 API，可以用来修改数据。你可能会问自己，“**为什么有两个可选的 API**？”或者“**我什么时候应该选择一个而不是另一个？**”。让我们揭开它们的神秘面纱，探索它们的异同，这样你就可以回答这些问题了。

本文中的代码示例引用了一个虚构的音乐行业数据库，其中有两个表**歌手**和**专辑**，模式如下。**专辑**是**歌手**的交叉表，其中…