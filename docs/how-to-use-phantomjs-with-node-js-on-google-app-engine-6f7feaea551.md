# 如何在 Google App Engine 上结合 Node.js 使用 PhantomJS

> 原文：<https://medium.com/google-cloud/how-to-use-phantomjs-with-node-js-on-google-app-engine-6f7feaea551?source=collection_archive---------1----------------------->

有一些帖子说你不能在 GAE(谷歌应用引擎)上使用 PhantomJS，但现在这不是真的。本文展示了如何通过 Node.js 配置使用 PhantomJS。

## 放弃

虽然下面的过程在写作时工作得很好，但这并不意味着它会工作很长时间。我甚至不确定这是否值得推荐。请理解风险。

## 背景

当你用 Node.js 编码的时候， [npm](https://www.npmjs.com) 上有很多包，大部分是用 JavaScript 写的，一部分是原生代码，还有一部分运行在单独的进程中。PhantomJS 是其中之一，它必须在与 Node.js 不同的进程中运行。PhantomJS 具有捕获网站的独特功能，目前还没有与 Node.js 很好集成的替代产品。

GAE 是一个 PaaS，通常仅限于运行一个流程。然而，相对较新的[灵活环境](https://cloud.google.com/appengine/docs/flexible/)(正式名称为“托管虚拟机”)更加灵活。它只是允许在谷歌计算引擎虚拟机的 Docker 容器中运行任何东西。

## 问题

Node.js 有一个很好的 PhantomJS 包，叫做[PhantomJS-预构建](https://www.npmjs.com/package/phantomjs-prebuilt)。如果你看看它的[依赖项](https://www.npmjs.com/browse/depended/phantomjs-prebuilt)，你会看到这么多的包依赖于它。现在，这个包在大多数情况下都能很好地工作，但是在文档中有一个重要的注意事项。

> 不需要安装 Qt、WebKit 或任何其他库。然而，它仍然依赖于 fontconfig(包 Fontconfig 或 libfontconfig，取决于发行版)。

问题是这个 *fontconfig* 包没有安装在 GAE 的 Node.js 的容器中。

## 解决办法

一个可能的解决方案是使用[定制运行时](https://cloud.google.com/appengine/docs/flexible/custom-runtimes/)并编写自己的 docker 文件来创建一个容器。

然而，我想要更简单的东西，并找到了解决方案。您只需要在 package.json 文件中添加一行。

```
{
  "scripts": {
    "preinstall": "apt-get update && apt-get -y 
install libfontconfig1"
  },
  "dependencies": {
    "phantomjs-prebuilt": "^2.1.7"
  }
}
```

请注意，它只显示了 package.json 文件的一部分。这很管用。

## 一些最后的想法

这里没什么好说的，但是我也不确定这是不是最好的方法。理想情况下，最好有一个与 PhantomJS 功能相似的纯 Node.js 包。对于最后一个评论，我要指出的是，您可能需要一些调整，以便*预安装*脚本只能在 GAE 上运行。