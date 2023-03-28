# 阿帕奇光束:测试

> 原文：<https://medium.com/google-cloud/apache-beam-testing-5684874afb51?source=collection_archive---------1----------------------->

为我们的管道编写测试是高效和有效的数据处理软件开发的重要一步。beam 中的单元测试通常使用 DirectRunner 完成，它包含在 Java 和 Python SDK 中。

**对 DoFns 和 PTransforms 进行单元测试**

为了能够执行这个测试，首先我们可以执行以下方法:

*   创建一个测试管道(请分别参考 [Python](https://beam.apache.org/releases/pydoc/2.29.0/apache_beam.testing.test_pipeline.html#apache_beam.testing.test_pipeline.TestPipeline) 和 [Java](https://beam.apache.org/releases/javadoc/2.29.0/org/apache/beam/sdk/testing/TestPipeline.html) )。
*   创建一个测试输入数据并使用它作为 Create(请分别参考 [Python](https://beam.apache.org/releases/pydoc/2.29.0/apache_beam.transforms.core.html#apache_beam.transforms.core.Create) 和 [Java](https://beam.apache.org/releases/javadoc/2.29.0/org/apache/beam/sdk/transforms/Create.html) 转换的输入来创建一个 PCollection
*   对输入 Pcollection 应用转换，并将其保存到另一个 PCollection
*   使用断言实用程序(请分别参考 Python 中 testing.util 包中的 [assert_that](https://beam.apache.org/releases/pydoc/2.29.0/apache_beam.testing.util.html#apache_beam.testing.util.assert_that) 和 Java 中的 [PAssert](https://beam.apache.org/releases/javadoc/2.29.0/org/apache/beam/sdk/testing/PAssert.html) )。

TestPipeline 是一个特殊的实用类，包含在 Beam SDK 中，用于测试转换和管道逻辑。当我们测试代码时，我们应该使用 TestPipeline 而不是 Pipeline 来创建 Pipeline 对象。Create transfrom 接受内存中的对象集合。目标是从我们的 PTransforms 中获得一小组测试输入数据，我们知道期望的输出 PCollection。Create transfrom 获取内存中的对象集合(一个 Java iterable ),并从该集合创建一个 PCollection。目标是从我们的转换中获得一小组测试输入数据，我们知道期望的输出 PCollection。

```
# Creating a TestPipeline in Python
with TestPipeline() as p:
    INPUTS = [fake_input_1, fake_input_2]
    test_output = p | beam.Create(INPUTS) | # Transforms to be tested// Creating a TestPipeline in Java
TestPipeline p = TestPipeline.create();
List<String> input = Arrays.asList(testInput);
outputPColl = p.apply(Create.of(input).apply(...);
```

最后我们可以用断言来比较结果和预期结果:

```
# Using assertion in Python
assert_that(test_output, equal_to(EXPECTED_OUTPUTS))// Using PAssert in Java
PAssert.that(outputPColl).containsInAnyOrder(expectedOutput);
```

**使用 TestStream 执行流处理逻辑测试**

当执行从测试流中读取的管道时，读取会等待每个事件的所有结果完成，然后再继续处理下一个事件，包括当处理时间提前且适当的触发器触发时。TestStream 允许触发的效果，并允许在管道上观察和测试延迟。这包括关于延迟触发器和由于延迟而丢弃的数据的逻辑。TestStream 类允许我们模拟实时消息流，同时控制处理时间和水印的进度。我们使用 InvervalWindow 类来定义我们希望检查的窗口，然后使用 assertion 来检查每个窗口的值(Python 使用 assert_that 和 equal_per_window，Java 使用 inWindow 和 PAssert)。