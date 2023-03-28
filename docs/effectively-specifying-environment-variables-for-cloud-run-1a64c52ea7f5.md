# 有效地指定云运行的环境变量

> 原文：<https://medium.com/google-cloud/effectively-specifying-environment-variables-for-cloud-run-1a64c52ea7f5?source=collection_archive---------3----------------------->

*(为了更好的阅读体验，* [*阅读我博客上的这篇文章*](https://ahmet.im/blog/mastering-cloud-run-environment-variables/) *)。)*

如今，许多微服务应用主要通过环境变量进行配置。如果您正在部署云运行，使用 CLI 指定许多环境变量可能看起来相当痛苦:

`--set-env-vars="**LOG_LEVEL=**verbose,**EXPERIMENTS=**ShowInactiveUsers,CountIncorrectLoginAttempts,"Test 1",**DB_CONN=**sqlserver://username@host/instance?param1=value&param2=value,**DB_PASS=**berglas://my-bucket/path/to/my-secret?destination=tempfile,**GODEBUG=**schedtrace=9000"`

不要害怕！有更好的方法。我已经在云运行文档中解释了所有这些[，但是这篇文章将会有一些讨论。](https://cloud.google.com/run/docs/configuring/environment-variables#command-line)

敏锐的读者可能会立即注意到上面的例子行不通:

*   `gcloud`希望使用逗号来拆分`K=V`对，尽管有些值中包含逗号
*   由`"`分隔的参数中有未转义的`"`字符，将导致 bash 解析失败
*   如果您在 bash 中运行它，它将在把它传递给`gcloud`命令之前去掉像`"..."`这样的部分。

所以在这篇文章中，我会给你一些建议来避免这种想法:

1.  不要将所有环境变量都塞进一个选项中。
2.  尽可能地防止您的 shell(例如 bash)解释参数。
3.  如果事情失去控制，用 YAML 切换到声明性资源模型。

# 1.环境变量太多？分开他们

当您第二次运行`gcloud run deploy`命令时，您有两种方法来指定环境变量:

1.  用`--set-env-vars`再次指定所有需要的环境变量。
2.  仅指定**要更新`--update-env-vars`的**环境变量，并使用`--remove-env-vars`取消设置您想要的环境变量。

我个人更喜欢第一个选项，因为我用完全部署命令为我的应用程序编写了一个`deploy.sh`。那么如何应对太多的环境变量呢？

云运行实际上允许您多次指定这些`--set-env-vars`或`--update-env-vars`选项**，因此您可以将上述声明拆分为多个参数，如下所示:**

```
$ gcloud run deploy \
    --set-env-vars LOG_LEVEL=verbose \
    --set-env-vars "EXPERIMENTS=IncorrectLoginAttempts,\"Test 1\"" \
    --set-env-vars GODEBUG=schedtrace=9000
    [...]
```

**那更好，但是它是傻瓜证明吗？不。正如你可能看到的，我们开始做一些丑陋的事情，如转义引用，还有更多隐藏的角落情况。**

# **2.防止对环境变量的误解**

**如果您将它传递给`gcloud`，它会将逗号(`,`)字符视为另一个环境变量声明，并尝试将该值拆分成多个环境变量:**

```
[...]
--set-env-var "A=B,C,D"
```

**这是由设计决定的，在此有[的解释。](https://cloud.google.com/sdk/gcloud/reference/topic/escaping)**

**为了防止用逗号分割，您需要指定一个不同的自定义分隔符，您确信它不会出现在您的值字符串中，例如`##`:**

```
[...]
--set-env-vars "^##^A=B,C,D"
```

**是的，这不是最好的解决办法，但会有用的。以下是在 shell 中运行时可能会给您带来麻烦的另外两个示例:**

*   **`--set-env-vars A="B C D"`:这不会保留引号，但是可以确保值中的空格不会被 shell 分割。**
*   **单引号通常是保存列表中内容的简单方法(但是如果你的值实际上只有一个单引号呢？-正确地逃避这些可能会很痛苦！)**
*   **`--set-env-vars 'A="B C D"'`:这将保留值中的双引号。**

# **3.从文件中读取环境变量**

**`gcloud run deploy`不支持`--env-vars-file`不像谷歌云功能或谷歌应用引擎部署命令。**

**然而，你实际上可以从 YAML 编写的清单文件中部署你的整个云运行服务。这是因为云运行服务实际上是可调用的 API 对象[，正如我在之前的文章](https://ahmet.im/blog/cloud-run-is-a-knative/)中解释的。**

**这是用`gcloud beta run services replace`命令完成的(希望它能很快升级到 beta → GA！)在这个 yaml 文件中，看一下`env`部分**

```
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: hello
spec:
  template:
    spec:
      containers:
      - image: gcr.io/google-samples/hello-app:1.0
        **env:**
        - name: LOG_LEVEL
          value: verbose
        # we can break strings over multiple lines
        # see: https://yaml-multiline.info/
        - name: EXPERIMENTS
          value: 'ShowInactiveUsers,
          CountIncorrectLoginAttempts,
          "Test 1"'
        - name: DB_CONN
          value: sqlserver://username@host/instance?param1=value&param2=value
        # [...]
```

**然后，您可以在每次想要更新服务时运行此命令:**

```
**gcloud beta run services replace** FILE.yaml
```

**这种方法有什么注意事项吗？是的，现在我们被 YAML 语法的规则所束缚。如果你有类似`value: 5`或`value: true`的东西，这些不会被当作`string` s，所以你需要明确地引用它们:**

```
value: "5"
value: "true"
```

**未能对这些进行转义将导致 API 出错。**

***原载于 2020 年 4 月 27 日*[*https://Ahmet . im*](https://ahmet.im/blog/mastering-cloud-run-environment-variables/)*。***