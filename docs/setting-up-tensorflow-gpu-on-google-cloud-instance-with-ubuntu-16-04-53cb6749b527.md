# 使用 Ubuntu 16.04 在 Google Cloud 实例上设置 TensorFlow GPU

> 原文：<https://medium.com/google-cloud/setting-up-tensorflow-gpu-on-google-cloud-instance-with-ubuntu-16-04-53cb6749b527?source=collection_archive---------0----------------------->

我最近建立了一个谷歌云实例来训练一些 TensorFlow 模型。虽然 Amazon EC2 的 ami 已经为您配置好了一切，但在 Google Cloud 上，您需要自己设置一切。我花了几天时间来做这件事，我在网上找到的说明——无论是从[谷歌](https://cloud.google.com/compute/docs/gpus/add-gpus#install-gpu-driver)还是从[其他地方](/google-cloud/using-a-gpu-tensorflow-on-google-cloud-platform-1a2458f42b0)都是针对旧版本的 TensorFlow 的，所以不能与最新版本一起使用，在我写这篇文章的时候，最新版本是 1.7。

这些指令将适用于 1.7 版，并且已经过多次测试。我希望它们能帮助人们不必花几天时间在网上搜索来解码各种错误信息。

这摘自 Google 说明，但使用了适用于 TensorFlow v1.7 版的正确版本的 CUDA:

```
curl -O [https://developer.download.nvidia.com/compute/cuda/repos/ubuntu1604/x86_64/cuda-repo-ubuntu1604_9.0.176-1_amd64.deb](https://developer.download.nvidia.com/compute/cuda/repos/ubuntu1604/x86_64/cuda-repo-ubuntu1604_9.0.176-1_amd64.deb)
sudo dpkg -i cuda-repo-ubuntu1604_9.0.176-1_amd64.deb
sudo apt-get update
sudo apt-get install cuda-9-0
sudo nvidia-smi -pm 1
```

然后，您应该验证所有组件都已安装，并且可以使用:

```
nvidia-smi
```

谷歌的说明没有提到安装 cudnn，但它似乎是必需的。要下载它，你需要注册 [Nvidia 的开发者程序](https://developer.nvidia.com/developer-program)，下载它，然后上传到你的实例。我用 SCP 上传了它，花了一些时间。我用的文件是 libcudnn 7 _ 7 . 0 . 4 . 31–1+cuda 9.0 _ amd64 . deb，这是用 TensorFlow 1.7 的 cuda-9.0 需要的版本。上传后，您可以使用以下软件进行安装:

```
sudo dpkg -i libcudnn7_7.0.4.31-1+cuda9.0_amd64.deb
```

一旦安装完毕，你需要设置一些路径变量。Google 的指令将变量临时添加到 path 中，因此需要在每次启动实例时运行。这将永久添加它们:

```
echo 'export CUDA_HOME=/usr/local/cuda' >> ~/.bashrc
echo 'export PATH=$PATH:$CUDA_HOME/bin' >> ~/.bashrc
echo 'export LD_LIBRARY_PATH=/usr/local/cuda/extras/CUPTI/lib64:$LD_LIBRARY_PATH' >> ~/.bashrcsource ~/.bashrc
```

最后，您可以安装 TensorFlow:

```
sudo apt-get install python3-dev python3-pip libcupti-dev
sudo pip3 install tensorflow-gpu
```

请注意，我使用的是 python3 和 pip3，但是如果您想使用 python3，只需从上面的命令中删除“3”即可。完成后，您应该能够正确无误地导入 tensorflow。

最后，Google 建议了一些设置来优化 GPU 性能:

```
# this applies to all GPUs
sudo nvidia-smi -pm 1# these only apply to Nvidia Tesla K80s
sudo nvidia-smi -ac 2505,875
sudo nvidia-smi --auto-boost-default=DISABLED
```

如果你在运行 TensorFlow 代码时仍然有问题，我[找到了](https://devtalk.nvidia.com/default/topic/936212/tensorflow-cannot-find-cudnn-ubuntu-16-04-cuda7-5-/)以下我执行的步骤，虽然我不确定它们是否必要。我这样做只是因为，但是我遇到的问题可以通过正确导出路径来解决，但是我不想设置另一个实例来检查它们是否是必需的:

1.  `cd /usr/local/cuda`
2.  `sudo ln -s /usr/lib/x86_64-linux-gnu/ lib64`
3.  `sudo ln -s /usr/include/ include`
4.  `sudo ln -s /usr/bin/ bin`
5.  `sudo ln -s /usr/lib/x86_64-linux-gnu/ nvvm`
6.  `sudo mkdir -p extras/CUPTI`
7.  `cd extras/CUPTI`
8.  `sudo ln -s /usr/lib/x86_64-linux-gnu/ lib64`
9.  `sudo ln -s /usr/include/ include`

这里的步骤摘自[这篇文章](/google-cloud/using-a-gpu-tensorflow-on-google-cloud-platform-1a2458f42b0)，这是我找到的最有用的说明，但对当前版本的 TensorFlow 无效。