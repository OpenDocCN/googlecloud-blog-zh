# 本地 pip 包和谷歌应用引擎

> 原文：<https://medium.com/google-cloud/local-pip-package-and-google-app-engine-c39b1e6412b0?source=collection_archive---------1----------------------->

谷歌的应用引擎沙箱是一个神奇的解决方案，可以快速创建应用程序。您已经获得了所需的一切，以及大量资源、样本和快速入门。

然而，在用 Python 开发多个服务时，您可能会发现应该为您的域、实用程序等创建至少一个公共库。

来自于。在网络世界中，私有的 NuGet feeds 对于组织来说是一件很平常的事情，我猜想 *pip* 也有同样的情况——尤其是因为 *pip* 已经存在了很长时间。

我错了。

我开始在谷歌上搜索，基本上只有简单的解决方案，比如:

> "使用 SimpleHTTPServer 在本地计算机上手动创建您的包并托管它"
> 
> "使用桶"
> 
> " $ pip 安装。"

我明白了，这就是开源精神——事情应该是开放的，所以如果我真的想让事情不对公众开放，也许我应该使用 Git 中的子模块特性来做到这一点。然而，问题是，我还不想公开这个应用的领域和特性。此外，我这样做是因为我真的想更多地了解 pip。

所以我开始将一些模块转移到我创建的新包中，并阅读了 [***这本简单的指南***](https://python-packaging.readthedocs.io/en/latest/minimal.html) ，我想将它推荐给需要了解 pip 的每个人。

使用这个指南，我学会了

```
1\. Create a setup.py with a simple module.
2\. Install the package locally using '$ pip install .'
3\. Install the package locally as a symlink '$ pip install -e .'
4\. Create an actual package using '$ python setup.py sdist'
```

理论上我已经控制了一切，我制作了我的模块，设法将它们打包在同一个名为' *myapp-shared* 的包中，我很高兴——运行 *sdist* 命令后的 dist-package 看起来不错。

然而，我很难让这个包与 *virtualenv* 和 GAE 运行时一起工作。

我创建了一个源代码发行版，并尝试使用指向 *dist* 文件夹位置的 pip 的 *find-links* 选项来安装该包，在 shell 中一切似乎都正常，但是当我更新共享包并再次运行 *sdist* 时，这似乎无法正常工作——当我查看 *lib* 文件夹时，我发现了旧的那个，就好像它刚刚从本地缓存安装的一样(？？？).

然而，经过多次反复之后，我发现最简单的方法是忽略 sdist-命令，只需从 GAE 项目的根目录运行这个命令:

```
pip install ../../../myapp-shared -t lib --upgrade
```

因此，我将这一行添加到现有的 install.sh 脚本中:

```
*#!/bin/sh* virtualenv env
source env/bin/activate
pip install -r requirements.txt -t lib
pip install ../../../myapp-shared -t lib --upgrade
```

所以最后，最简单的解决方案是最好的。

干杯。