# 使用谷歌云机器学习服务服务 Keras 模型

> 原文：<https://medium.com/google-cloud/serve-keras-models-using-google-cloud-machine-learning-services-910912238bf6?source=collection_archive---------3----------------------->

Google Cloud ML 使您能够在云中训练您的模型并提供在线/批量预测。我已经找到了许多教程和代码示例，但没有一个是完整的。

如果你只是在云中训练你的模型，只要你按照 Google 提供的[操作指南中的步骤去做，你在那个阶段应该不会有任何问题。我强烈建议您看一看](https://cloud.google.com/ml-engine/docs/getting-started-training-prediction)[“使用 Keras 使用人口普查收入数据集预测收入”](https://github.com/GoogleCloudPlatform/cloudml-samples/tree/master/census/keras)示例，并关注如何使用 TensorBoard 实现可视化培训结果。

Tensorflow 是使用 [Tensorflow Serving](https://github.com/tensorflow/serving) 库在 Google Cloud ML 上提供预测的主要框架。除了可用的 Google [操作指南](https://cloud.google.com/ml-engine/docs/deploying-models)，以下是我鼓励你在部署第一个模型时使用的重要资源:

1.  查看本教程: [Keras 作为 TensorFlow 的简化接口](https://blog.keras.io/keras-as-a-simplified-interface-to-tensorflow-tutorial.html)描述 Keras 层与 Tensorflow tensors 的兼容性。特别注意**第 4 节**:导出一个带 TensorFlow-serving 的模型。
2.  仔细阅读 Census Keras 示例代码。这是一个很好的参考，并提供了一个端到端的例子(训练到预测)，但调用 set_learning_phase **很重要，否则预测将失败**，并显示 StatusCode。INVALID_ARGUMENT 错误( [**刚刚用此修复**](https://github.com/GoogleCloudPlatform/cloudml-samples/pull/141) 创建了一个 pull 请求)。

以下是您在开发任务时必须注意的主要构建模块:

1.  编译一个 Keras 模型(见上面 set_learning_phase)。
2.  可以使用 tensorflow.python.lib.io 将文件写入 GCS，但是您必须使用 GCS python 客户端库才能与 GCS 存储桶进行交互。
3.  使用 TensorBoard 回调来可视化您的训练过程。
4.  训练 Keras 模型。
5.  将 Keras 模型导出到一个[tensor flow Serving saved model](https://www.tensorflow.org/programmers_guide/saved_model)。
6.  创建一个模型实例，例如:*g cloud ml-engine models create model name—regions us-central 1*
7.  基于您导出的模型实例创建模型版本($MODEL_BINARIES 是您的 GS://exported MODEL location):*g cloud ml-engine versions create v1—MODEL MODEL name—origin $ MODEL _ BINARIES—runtime-version 1.4*
8.  通过运行在线预测来测试您的模型:*g cloud ml-engine predict-model model name-JSON-instances model/request . JSON-version v1*

*   确保您的 request.json 结构与您提供给 SavedModel builder 的签名一致。
*   如有错误，请仔细阅读。通常，该错误具有很强的描述性，例如以下关于所提供的输入形状的内容:

{

" error ":"预测失败:模型执行期间出错:AbortionError(code=StatusCode。INVALID_ARGUMENT，details=\ "输入必须是四维的[1，1，1，150，150，3]\ n \ T[[Node:Conv2D _ 1/convolution = Conv2D[T = DT _ FLOAT，_output_shapes=[？，148，148，32]]，data_format=\"NHWC\ "，padding=\ "有效\ "，strides=[1，1，1，1]，use_cudnn_on_gpu=true，_ device = \ "/job:localhost/replica:0/task:0/device:CPU:0 \ "](_ arg _ conv2d _ 1 _ input _ 0 _ 0，conv2d_1/kernel/read)])"

}

或者这个(与 set_learning_phase 相关):

NetworkError(code=StatusCode。INVALID_ARGUMENT，details= "您必须为 dtype uint 8
的占位符张量' keras_learning_phase '提供一个值[[Node:keras _ learning _ phase = Placeholder[dtype = DT _ uint 8，shape=[]，_ device = "/job:localhost/replica:0/task:0/CPU:0 "]()]]")

祝你好运！