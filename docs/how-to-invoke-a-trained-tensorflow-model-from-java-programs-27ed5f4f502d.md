# 如何从 Java 程序中调用经过训练的张量流模型

> 原文：<https://medium.com/google-cloud/how-to-invoke-a-trained-tensorflow-model-from-java-programs-27ed5f4f502d?source=collection_archive---------0----------------------->

创建和训练 TensorFlow 机器学习模型的主要语言是 Python。然而，许多企业服务器程序是用 Java 编写的。因此，您经常会遇到需要从 Java 程序调用用 Python 训练的 Tensorflow 模型的情况。

如果你在谷歌云平台上使用 [Cloud ML](https://cloud.google.com/ml/) 引擎，这没有问题——在 Cloud ML 引擎中，预测是通过 REST API 调用进行的，因此你可以从任何编程语言进行预测。但是，如果您已经下载了 TensorFlow 模型，并希望离线进行预测，该怎么办？

以下是如何使用 Python 中训练的 Tensorflow 模型在 Java 中进行预测。

> **注意:Tensorflow 团队现在已经开始添加 Java 绑定。详见**[https://github . com/tensor flow/tensor flow/tree/master/tensor flow/Java](https://github.com/tensorflow/tensorflow/tree/master/tensorflow/java)**。先试试那个，如果对你不起作用，来这里...**

## 用 Python 写出模型文件

首先要做的是以两种格式保存 Python 中的 TensorFlow 模型:(a)权重、偏差等。作为一个“saver_def”文件(b)图形本身作为一个 protobuf 文件。为了保持理智，您可能希望将图形保存为文本和二进制 protobuf 格式。您会发现通读文本格式有助于找到 TensorFlow 分配给未明确分配名称的节点的名称。从 Python 编写这三个文件的代码:

```
# create a Saver object as normal in Python to save your variables
saver = tf.train.Saver(...)# Use a saver_def to get the "magic" strings to restore
saver_def = saver.as_saver_def()
print saver_def.filename_tensor_name
print saver_def.restore_op_name# write out 3 files
saver.save(sess, 'trained_model.sd')
tf.train.write_graph(sess.graph_def, '.', 'trained_model.proto', as_text=False)
tf.train.write_graph(sess.graph_def, '.', 'trained_model.txt', as_text=True)
```

在我的例子中，save_def 输出的两个神奇字符串是 *save/Const:0* 和*save/restore _ all*——所以这就是我的 Java 代码中看到的内容。如果您的代码不同，那么在编写 Java 代码时更改它们。

的。sd 文件包含权重、偏差等。(图表中变量的实际值)。的。原型文件是一个二进制文件，包含您的计算图和。txt 对应的文本版本。

## 从 Java 调用 Tensorflow C++

即使您可能已经在 Python 中使用 Tensorflow 向模型提供数据并对其进行训练，Tensorflow Python 包实际上调用 C++实现来执行实际工作。因此，我们可以使用 Java 本地接口(JNI)直接调用 C++，并使用 C++创建图形，从 Java 中恢复模型的权重和偏差。

不用手工编写所有的 JNI 调用，可以使用一个名为 [JavaCpp](https://github.com/bytedeco/javacpp-presets/) 的开源库来完成这项工作。要使用 JavaCpp，请将这个依赖项添加到 Java Maven pom.xml 中:

```
<dependency>
  <groupId>org.bytedeco.javacpp-presets</groupId>
  <artifactId>tensorflow</artifactId>
  <version>0.9.0–1.2</version>
</dependency>
```

如果您正在使用一些其他的构建管理系统，将 tensorflow *的 Javacpp 预置及其所有依赖项*添加到您的应用程序的类路径中。

## 用 Java 创建模型

在您的 Java 代码中，读取 proto 文件来创建一个图形定义，如下所示(为了清楚起见，省略了导入):

```
final Session session = new Session(new SessionOptions());
GraphDef def = new GraphDef();
tensorflow.ReadBinaryProto(Env.Default(), 
                           "somedir/trained_model.proto", def);
Status s = session.Create(def);
if (!s.ok()) {
    throw new RuntimeException(s.error_message().getString());
}
```

接下来，使用 Session::Run()从保存的模型文件中恢复权重和偏差。注意 saver_def 中的神奇字符串是如何使用的。

```
// restore
Tensor fn = new Tensor(tensorflow.DT_STRING, new TensorShape(1));
StringArray a = fn.createStringArray();
a.position(0).put(“somedir/trained_model.sd”); 
s = session.Run(new StringTensorPairVector(new String[]{“save/Const:0”}, new Tensor[]{fn}), new StringVector(), new StringVector(“save/restore_all”), new TensorVector());
if (!s.ok()) {
   throw new RuntimeException(s.error_message().getString());
}
```

## 用 Java 做预测

此时，您的模型已经准备好了。你现在可以用它来做预测。这类似于在 Python 中的做法——必须为所有占位符传入值，并计算输出节点。不同之处在于，您必须知道占位符和输出节点的实际*名称*。如果在 Python 中没有为这些节点指定唯一的名称，Tensorflow 会为它们指定名称。您可以通过查看写出的 trained_model.txt 文件来找出它们是什么。或者，您可以返回到 Python 代码，为您记住的关键节点分配名称。在我的例子中，输入占位符被称为占位符；丢弃节点占位符称为 Placeholder_2，输出节点称为 Sigmoid。您将在下面的 Session::Run()调用中看到这些引用。

在我的例子中，神经网络使用 5 个预测变量。假设我有一组输入，这些输入是我的神经网络模型的预测器，并希望对 2 组这样的输入进行预测，我的输入是一个 2×5 矩阵。我的 NN 只有一个输出，所以对于 2 组输入，输出张量是一个 2x1 矩阵。丢弃节点的硬编码输入为 1.0(在预测中，我们保留所有节点—丢弃概率仅用于训练)。所以，我有:

```
// try to predict for two (2) sets of inputs.
Tensor inputs = new Tensor(
         tensorflow.DT_FLOAT, new TensorShape(2,5));
FloatBuffer x = inputs.createBuffer();
x.put(new float[]{-6.0f,22.0f,383.0f,27.781754111198122f,-6.5f});
x.put(new float[]{66.0f,22.0f,2422.0f,45.72160947712418f,0.4f});Tensor keepall = new Tensor(
        tensorflow.DT_FLOAT, new TensorShape(2,1));
((FloatBuffer)keepall.createBuffer()).put(new float[]{1f, 1f});TensorVector outputs = new TensorVector();// to predict each time, pass in values for placeholders
outputs.resize(0);
s = session.Run(new StringTensorPairVector(new String[] {“Placeholder”, “Placeholder_2”}, new Tensor[] {inputs, keepall}),
 new StringVector(“Sigmoid”), new StringVector(), outputs);
if (!s.ok()) {
   throw new RuntimeException(s.error_message().getString());
}// this is how you get back the predicted value from outputs
FloatBuffer output = outputs.get(0).createBuffer();
for (int k=0; k < output.limit(); ++k){
   System.out.println(“prediction=” + output.get(k));
}
```

就是这样——你现在使用 Java 来实现你的预测。有几个步骤，但是当一个人混合 3 种编程语言(Python、C++和 Java)时，这是预料之中的。但重要的是，这是可以做到的，而且相对简单。

当然，这样做并没有利用硬件加速和分布的优势。如果你想实时以非常高的速度进行预测，你应该考虑使用云 ML 引擎。