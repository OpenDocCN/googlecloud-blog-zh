# 谷歌云构建的代码覆盖率

> 原文：<https://medium.com/google-cloud/code-coverage-on-google-cloud-build-7fcf1b67f182?source=collection_archive---------1----------------------->

当您为您的项目设置 CI/CD 管道时，您可能想要计算代码覆盖率，并将其发送给第三方服务，如 Codecov。

我将向您展示如何在 [Google Cloud Build](https://cloud.google.com/cloud-build/) 上进行设置。

![](img/8c0c638e2fcadaadb0a038a4f4a8d95d.png)

# 问题

默认情况下， [codecov 工具](https://www.npmjs.com/package/codecov)必须从 git 存储库中运行，否则它会抛出一个错误:

```
fatal: Not a git repository
```

原因是 codecov 被设计为报告代码的特定分支和提交的覆盖率。

因为 [Google Cloud Build GitHub 应用](https://github.com/marketplace/google-cloud-build)不会重新创建一个完整的 git 存储库，所以我们需要一个变通办法。

# 解决办法

解决方案是手动传递分支并提交到工具:

```
$ codecov --token=X --disable=detect --commit=FOO --branch=BAR
```

云构建通过[内置替换](https://cloud.google.com/cloud-build/docs/configuring-builds/substitute-variable-values)使这种信息对构建可用。

## *cloudbuild.yaml*

```
steps: 
- name: 'gcr.io/cloud-builders/npm'  
  args: ['install'] - name: 'gcr.io/cloud-builders/npm'  
  args: ['run', 'ci'] - name: 'gcr.io/cloud-builders/npm'  
  args: 
   [
     'run', 
     'codecov',
     '--', 
     '--disable=detect', 
     '--commit=$COMMIT_SHA', 
     '--branch=$BRANCH_NAME'
   ]
```

## package.json

```
{
  "devDependencies": {
    "codecov": "3.1.0", 
    "mocha": "5.2.0",
    "nyc": "13.1.0"
  }, "scripts": {
     "ci": "NODE_ENV=test node_modules/.bin/nyc node_modules/.bin/mocha --reporter spec 'server/test/**/*test.js' && node_modules/.bin/nyc report --reporter=text-lcov > coverage.lcov",
     "codecov": "node_modules/.bin/codecov --token=X"
  }}
```

# 结论

您可以绕过 codecov 工具检测，手动传递当前 CI 运行的提交和分支。

云构建配置易于阅读:它安装依赖项，运行测试，然后将结果上传到 Codecov。

在我的下一篇文章中，我将向你展示如何缓存第一步以获得更好的性能。