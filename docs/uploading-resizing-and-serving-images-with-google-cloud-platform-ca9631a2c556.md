# 使用谷歌云平台上传、调整大小和提供图像

> 原文：<https://medium.com/google-cloud/uploading-resizing-and-serving-images-with-google-cloud-platform-ca9631a2c556?source=collection_archive---------0----------------------->

如今，越来越多的应用程序要求用户上传照片。从更新个人资料照片这样的简单事情，到 Snapchat 这样你可以分享一堆照片的更复杂的服务。这些应用程序运行在各种具有不同分辨率和不同网络条件的设备上。为了做到这一点，我们必须尽可能快地交付图像，并考虑到目标设备和屏幕分辨率，以最佳质量提供图像。

一种选择是自己创建一个图像服务，你可以上传图像，存储它们，并可能调整它们的大小。不幸的是，这样做通常在 CPU、存储、带宽方面非常昂贵，最终可能非常昂贵。这也是一项相当复杂的任务，许多事情都可能出错。

不过，通过使用[谷歌应用引擎](https://cloud.google.com/appengine/)和[谷歌云存储](https://cloud.google.com/storage/)，你可以通过使用它们的 API 轻松完成这个看似困难的任务。从[完成一个关于如何上传文件到云中的简单教程](https://cloud.google.com/appengine/docs/python/blobstore/)开始，如果你想看到它的运行，理解为什么它是有史以来最酷的事情之一，请阅读其余部分。

## 该功能

App Engine API 有一个非常有用的功能，可以提取一个 *magic* URL，用于在上传到云存储时提供图像:

> [**get _ serving _ URL()**](https://cloud.google.com/appengine/docs/python/images/functions#Image_get_serving_url)
> 
> 返回一个 URL，该 URL 以允许动态调整大小和裁剪的格式提供图像，因此您不需要在服务器上存储不同大小的图像。高度优化的无 cookieless 基础设施为映像提供**低延迟**。

实际上，这与谷歌用于谷歌照片等自己服务的基础设施是一样的。*神奇的*网址通常有以下形式:[**http://lh3.googleusercontent.com/93u...DQg**](http://lh3.googleusercontent.com/93uhV8K2yHkRuD63KJxlTi7SxjHS8my2emuHmGLZxEmX99_XAjTN3c_2zmKVb3XQ5d8FEkwtgbGjyYpaDQg)

## 关于这个神奇网址的一些细节

*   默认情况下，它返回最大长度为**512 像素** ( [链接](http://lh3.googleusercontent.com/93uhV8K2yHkRuD63KJxlTi7SxjHS8my2emuHmGLZxEmX99_XAjTN3c_2zmKVb3XQ5d8FEkwtgbGjyYpaDQg))的图像
*   通过将 **=sXX** 附加到它的末尾，其中 XX 可以是在**0–2560**范围内的任何整数，这将导致在不影响原始纵横比([链接=s256](http://lh3.googleusercontent.com/93uhV8K2yHkRuD63KJxlTi7SxjHS8my2emuHmGLZxEmX99_XAjTN3c_2zmKVb3XQ5d8FEkwtgbGjyYpaDQg=s256) )的情况下将图像缩小到最长尺寸
*   通过追加 **=sXX-c** ，该图像的裁剪版本将作为响应返回( [link=s400-c](http://lh3.googleusercontent.com/93uhV8K2yHkRuD63KJxlTi7SxjHS8my2emuHmGLZxEmX99_XAjTN3c_2zmKVb3XQ5d8FEkwtgbGjyYpaDQg=s400-c) )
*   通过追加 **=s0** (s 零)，原始图像被返回而没有任何调整大小或修改( [link=s0](http://lh3.googleusercontent.com/93uhV8K2yHkRuD63KJxlTi7SxjHS8my2emuHmGLZxEmX99_XAjTN3c_2zmKVb3XQ5d8FEkwtgbGjyYpaDQg=s0) )

![](img/953dcfbcd403d2bec3d67824583dbc69.png)

[http://lh3.goo…DQg](http://lh3.googleusercontent.com/93uhV8K2yHkRuD63KJxlTi7SxjHS8my2emuHmGLZxEmX99_XAjTN3c_2zmKVb3XQ5d8FEkwtgbGjyYpaDQg)

![](img/13f02d5a4fb906b88f3ced8f7ef567af.png)

[http://lh3 . goo…aDQg = s256](http://lh3.googleusercontent.com/93uhV8K2yHkRuD63KJxlTi7SxjHS8my2emuHmGLZxEmX99_XAjTN3c_2zmKVb3XQ5d8FEkwtgbGjyYpaDQg=s256)

![](img/b50cf085817eb5775d91739dc87fe092.png)

[http://lh3 . goo…DQg = s256-c](http://lh3.googleusercontent.com/93uhV8K2yHkRuD63KJxlTi7SxjHS8my2emuHmGLZxEmX99_XAjTN3c_2zmKVb3XQ5d8FEkwtgbGjyYpaDQg=s256-c)

如果这些还不足以满足开箱即用的需求，那么使用谷歌的神奇网址时，调整图片大小和缓存图片也是免费的。是的，你没看错，这是一项**免费服务**，你**只需为原始图像的实际存储**付费。

## 摘要

今天，你不必担心为你的下一个大创意提供图片。您只需上传一次图片，提取神奇的 URL，然后根据环境更新参数，直接在客户端使用它。类似 Snapchat 甚至更复杂的原型应用可以在一个周末内实现。

## 你自己试试

除了文档中已经提到的[教程](https://cloud.google.com/appengine/docs/python/blobstore/)之外，您还可以玩一个关于 [gae-init-upload](http://upload.gae-init.com/resource/upload/) 的实例(它基于开源项目 [gae-init](https://github.com/gae-init/gae-init) )。

## 信用

示例中的照片由 [Aleksandra Kiebdoj](https://www.instagram.com/p/BD9ZvurCSyC/) 拍摄。