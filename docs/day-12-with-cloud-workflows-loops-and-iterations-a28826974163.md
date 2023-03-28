# 云工作流第 12 天:循环和迭代

> 原文：<https://medium.com/google-cloud/day-12-with-cloud-workflows-loops-and-iterations-a28826974163?source=collection_archive---------2----------------------->

在这个云工作流系列的前几集中，我们已经了解了用于在步骤之间移动的[变量赋值](http://glaforge.appspot.com/article/day-3-with-cloud-workflows-variable-assignment-and-expressions)、和[切换条件等数据结构](http://glaforge.appspot.com/article/day-4-with-cloud-workflows-jumping-with-switch-conditions)，以及用于执行一些计算的[表达式](http://glaforge.appspot.com/article/day-3-with-cloud-workflows-variable-assignment-and-expressions)，其中可能包括一些内置函数。

有了所有这些之前的知识，我们现在有了所有的工具来创建循环和迭代，例如，迭代一个数组的元素，也许用不同的参数调用一个 API 几次。所以让我们来看看如何创建这样的迭代！

首先，让我们准备一些变量赋值:

```
- define:
    assign: 
        - array: ['Google', 'Cloud', 'Workflows']
        - result: ""
        - i: 0
```

*   数组变量将保存我们将要迭代的值。
*   结果变量包含一个字符串，我们将把数组中的每个值追加到这个字符串中。
*   I 变量是一个索引，用来知道我们在数组中的位置。

接下来，就像在编程语言的 for 循环中一样，我们需要为循环的结束准备一个条件。我们将在一个专门的步骤中完成:

```
- checkCondition:
    switch:
        - condition: ${i < len(array)}
          next: iterate
    next: returnResult
```

我们使用内置的 len()函数定义了一个开关，其中的条件表达式将当前索引位置与数组长度进行比较。如果条件为真，我们将进入迭代步骤。如果为 false，我们将转到结束步骤(这里称为 returnResult)。

让我们处理迭代体本身。这里非常简单，因为我们只是给变量赋值:我们将数组的第 I 个元素添加到结果变量中，然后将索引增加 1。然后，我们返回到 checkCondition 步骤。

```
- iterate:
    assign:
        - result: ${result + array[i] + " "}
        - i: ${i+1}
    next: checkCondition
```

请注意，如果我们要做一些更复杂的事情，例如用数组的元素作为参数调用 HTTP 端点，我们将需要两个步骤:一个用于实际的 HTTP 端点调用，一个用于递增索引值。然而，在上面的例子中，我们只对变量赋值，所以我们在这个简单的赋值步骤中完成了整个迭代。

当执行 checkCondition 步骤时，如果不满足条件(即我们已经到达数组的末尾)，然后我们被重定向到 returnResult 步骤:

```
- returnResult:
    return: ${result}
```

最后一步只是返回结果变量的值。

*原载于*[*http://glaforge.appspot.com*](http://glaforge.appspot.com/article/day-12-with-cloud-workflows-loops-and-iterations)*。*