# AI 平台笔记本“无头”训练

> 原文：<https://medium.com/google-cloud/ai-platform-notebooks-headless-training-b8efd905dfaf?source=collection_archive---------1----------------------->

## 比谷仓院子里的鸡更不容易引发

[AI 平台笔记本](https://cloud.google.com/ai-platform-notebooks)提供托管的 JupyterLab 笔记本实例，这是一个熟悉的工具，用于实验、开发和将模型部署到生产中。

缺失的部分是生产培训模型；这通常比最初的实验或产量预测要长得多。您进行实验的交互式笔记本可能不适合这样做:浏览器会话超时，连接丢失，图像渲染冻结，实例大小不适合训练。

本文主要关注以无头方式运行培训的两种方法:

*   [AI Hub](https://cloud.google.com/ai-hub)JupyterLab VM:paper mill
*   [AI 平台培训](https://cloud.google.com/ai-platform/training/docs/training-jobs)

# 人工智能中心 JupyterLab 虚拟机:造纸厂

[Papermill](https://papermill.readthedocs.io/en/latest/) 是一款以多种不同方式[参数化和执行 Jupyter 笔记本的工具](https://cloud.google.com/blog/products/ai-machine-learning/let-deep-learning-vms-and-jupyter-notebooks-to-burn-the-midnight-oil-for-you-robust-and-automated-training-with-papermill)，包括启动虚拟机。

其中之一是从笔记本虚拟机的命令行提交笔记本:这利用了您已经用 AI 库配置的自定义虚拟机配置，同时绕过了与在笔记本 UI 中呈现结果相关联的潜在冻结问题。笔记本电脑虚拟机还预装了 Papermill，因此无需额外配置。

*   如果你想使用 Papermill 在单独的虚拟机上运行，[这个位于 TPU 的 Next 2019 演讲](https://cloud.withgoogle.com/next/sf/sessions?session=MLAI212)将带你了解这一场景。

您可以从虚拟机的[云控制台- >计算引擎- >虚拟机实例](http://cloud.google.com/console/compute/instances) - > SSH 选项访问命令行，也可以从笔记本本身访问命令行；前者排除了笔记本用户界面的潜在问题。您可以在浏览器窗口中打开 SSH 会话，或者在专用客户端中打开 SSH 会话以获得额外的状态持久性。

## 参数化

您可以[参数化](https://papermill.readthedocs.io/en/latest/usage-parameterize.html)您的笔记本，以便在运行时将可变信息(如纪元数)传递到笔记本中。在标有“参数”关键字的单元格中设置默认参数，并在训练或其他步骤中引用这些参数；无论您在命令行上指定什么，这些都会被覆盖。

例如，带有 epochs 参数的笔记本的命令行:

## 执行

运行该命令时，指定一个输出笔记本；所有中间结果、记录等。都记录在这里。包括[日志输出标志](https://papermill.readthedocs.io/en/latest/usage-cli.html)以将笔记本输出写入 stderr(即终端窗口)

请注意，如果终端关闭，这会终止 SSH 会话，默认情况下，会终止在其中运行的任何笔记本，包括无头笔记本。

**如果终端关闭，继续执行**

要在终端会话关闭时继续执行，请使用如下命令序列之一:

这些命令显示进程 id (pid ),并将 stderr 和 stdout 分别重定向到 nohup.out 或~/output.txt。

您可以从 output 记事本监视正在运行的进程，也可以使用以下命令从新的终端会话监视正在运行的进程:

请注意，重定向 stdout 和 stderr 会引发以下错误；这不会影响笔记本执行或日志记录:

*   *属性错误:“NoneType”对象没有属性“send _ multipart”*

更多背景信息请点击此处:

*   [终端关闭后继续运行的命令](https://unix.stackexchange.com/questions/4004/how-can-i-run-a-command-which-will-survive-terminal-close#4006)
*   【nohup、disown 和&T11 的区别】

# 人工智能平台培训

这种方法是基于创建一个 Docker 映像，在基础 Tensorflow 映像之上分层放置您的库，然后将其部署到 [AI 平台培训](https://cloud.google.com/ai-platform/training/docs/training-jobs)；您还可以在本地运行映像进行测试。这样做的好处是，能够以不同于笔记本电脑环境的方式调整培训环境的规模，并在不影响作业执行的情况下自然关闭终端会话；您可以在[云控制台](http://cloud.google.com/console/ai-platform/jobs)中跟踪作业执行情况；日志被写入 Stackdriver。

将模型权重存储在 Google 云存储中，作为训练和预测步骤之间的链接；注意，你也可以[保存整个模型](https://www.tensorflow.org/beta/guide/keras/saving_and_serializing#whole-model_saving)，而不仅仅是[权重](https://www.tensorflow.org/guide/keras/save_and_serialize#weights-only_saving)；所选择的方法将取决于您的用例。

## 参数化

如前所述，您可以参数化执行，例如，用历元数。

## 样本脚本

## 执行

在您的虚拟机上逐步完成上面概述的步骤，或者作为 bash 脚本运行所有步骤。您需要在运行 VM 的项目中创建自己的 GCS bucket 如果您适当地配置服务帐户，这个 GCS 存储桶可以位于不同的虚拟机上。

如果您选择运行可选的本地培训测试步骤，您的虚拟机将需要一个 GPU。

## 检查站

为了在作业期间支持更好的故障恢复，您可以实现[检查点](https://www.tensorflow.org/beta/guide/checkpoints)；这个 Keras 示例演示了对历元的检查点操作。