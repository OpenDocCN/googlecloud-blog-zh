# 使用云构建器检查 JavaScript

> 原文：<https://medium.com/google-cloud/checking-javascript-with-cloud-builder-dab387f6d6ff?source=collection_archive---------0----------------------->

这篇文章展示了使用 Google [云构建](https://cloud.google.com/cloud-build/docs/)和 JavaScript linter [eslint](https://eslint.org/) 进行样式检查。随着 ES6 的发布和现代 JavaScript 的普及，您将需要集成 Node.js 和相关工具来开发任何实质性的 web 应用程序，不管您的后端语言是什么。检查 JavaScript 代码风格是开始集成的好地方。

[对应要点](https://gist.github.com/alexamies/8e7aa81d1078ceadc2298e0df92f9865)中的 sum.js 文件改编自 Mark Ethan Trostler 所著的《可测试的*JavaScript:确保可靠的代码* 一书，该书包含了对 JavaScript 代码风格和开发可测试 JavaScript 的精彩讨论。文件 sum.js 包含一些样式错误，我们希望使用 eslint 来识别和纠正这些错误。此外，我们可以使用
[Docker Cloud Builder](https://github.com/GoogleCloudPlatform/cloud-builders/tree/master/docker)运行 eslint，以避免在本地环境中设置许多 Node.js 依赖项，并建立一个可重复的自动化构建过程。

首先，将 Gist 文件复制到您的本地环境中。

我们可以通过使用 npm 安装 eslint 并使用命令运行，在本地进行样式检查

```
npm install -g eslint eslint-config-google
eslint .
```

这使用 Google 风格的规则，您可以通过编辑 eslint 配置文件. eslintrc.json 来更改这些规则。

要在云构建中运行，请使用以下命令

```
gcloud builds submit . --config=cloudbuild.yaml
```

我们预计会看到一些错误，因为 sum.js 有已知的问题。eslint 报告的缺少分号和缺少 [JSDoc](http://usejsdoc.org/) 的一些问题。通过编辑 sum.js 来纠正这些问题，用以下代码替换现有内容:

```
/**
 * Add two numbers together
 * @param {number} a - The first operand
 * @param {number} b - The second operand
 * @return {number} The sum of the two operands
 */
function sum(a, b) {
  return a + b;
}
console.log('1 + 1 = ' + sum(1, 1));
```

然后再次运行 gclould build

```
gcloud builds submit . --config=cloudbuild.yaml
```

这一次构建应该会成功。